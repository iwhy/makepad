# `ui_runner.rs` — 跨线程 UI 异步执行器

`UiRunner<T>` 提供了一种安全、类型化的机制，允许从非 UI 线程（如 tokio 异步任务、网络回调、计算密集型工作线程）调度闭包回到 UI 线程执行。它是 Makepad 框架中处理跨线程通信的核心基础设施。

---

## `UiRunner<T>`

```rust
pub struct UiRunner<T> {
    key: usize,
    target: PhantomData<fn() -> T>,
}
```

### 设计原理

- **`key`**：一个 usize 值，用于区分来自不同 `UiRunner` 的动作。同一应用中可能有多个 `UiRunner` 实例（如不同窗口或不同组件），每个使用不同的 key。全局 `Cx::post_action` 使用 `ActionTrait` 的分发机制，`handle` 方法通过比较 key 来筛选属于当前 runner 的动作。
- **`PhantomData<fn() -> T>`**：使用函数指针类型而非直接使用 `T`，这是因为：
  1. `fn() -> T` 是 `Send` 和 `Sync` 的（只要 `T` 满足相应约束），而直接使用 `T` 会对类型参数施加不必要的 trait 约束。
  2. 这种惯用法（见 PhantomData 文档）用于"拥有"类型参数而不实际持有该类型的值。

### `UiRunner::new(key)`
- 创建一个新的 `UiRunner`，使用指定的 key 标识。建议使用 `ui_runner()` 方法（如 `your_widget.ui_runner()` 或 `your_app_main.ui_runner()`）而非直接构造。

### `UiRunner::handle(self, cx, event, scope, target)`
- 在 UI 线程的事件处理中调用，用于处理所有先前通过 `defer` 调度的动作。
- 接收 `Event::Actions` 事件，遍历所有的 `Action`，查找类型为 `UiRunnerAction<T>` 且 key 匹配的动作。
- 每个匹配到的动作从其 `Mutex<Option<Box<dyn DeferCallback<T>>>>` 中取出闭包执行，传入 `target`、`cx`、`scope`。
- 闭包执行后即被 `take()` 移除，因此每个延迟动作只执行一次。
- 典型用法是在 `handle_event` 中调用：`self.ui_runner().handle(cx, event, scope, self);`。

### `UiRunner::defer(self, f)`
- 从任意线程调用，将闭包 `f` 调度到 UI 线程执行。
- 实现方式：
  1. 将闭包包装为 `UiRunnerAction<T>`，闭包存入 `Mutex<Option<...>>` 中，与当前 runner 的 key 绑定。
  2. 通过 `Cx::post_action(action)` 将动作全局投递。`post_action` 使用一个全局的 `ACTION_SENDER_GLOBAL` Mutex 保护的单向通道。
  3. 动作投递后，以信号量（`SignalToUI::set_action_signal()`）唤醒 UI 线程的事件循环。
- 该方法是非阻塞的：立即返回，闭包在 UI 线程处理到该动作时异步执行。

### `UiRunner::block_on(self, f)`
- 类似于 `defer`，但会**阻塞当前线程**直到 UI 线程执行完闭包并返回结果。
- 实现方式：
  1. 创建一个 `std::sync::mpsc::channel`。
  2. 通过 `defer` 调度一个闭包，该闭包执行用户逻辑并通过 `tx.send(result)` 返回结果。
  3. 当前线程在 `rx.recv()` 上阻塞等待。
- 返回值 `R` 必须满足 `Send + 'static` 约束，以便能跨线程传递。
- 文档警告：不要在 tight loop 中使用此方法，因为 UI 线程可能正忙，导致长时间阻塞。

---

## `UiRunnerAction<T>`

```rust
struct UiRunnerAction<T> {
    f: Mutex<Option<Box<dyn DeferCallback<T>>>>,
    key: usize,
}
```

- `UiRunnerAction` 是一个实现了 `ActionTrait` 的私有消息结构体（借助全局 `Trait` 实现，通过 `ActionTrait` 在类型系统中注册）。
- `f` 使用 `Mutex` 保护，因为 `post_action` 可能需要 `Send` 约束（从另一个线程调用）。
- `key` 字段用于 `handle` 方法中筛选正确的动作。

### `Debug` 实现
- 不输出闭包内容（`"...""`），只显示 key，避免闭包无法实现 `Debug` 的问题。

---

## `DeferCallback<T>` Trait

```rust
pub trait DeferCallback<T>: FnOnce(&mut T, &mut Cx, &mut Scope) + Send + 'static {}
impl<T, F: FnOnce(&mut T, &mut Cx, &mut Scope) + Send + 'static> DeferCallback<T> for F {}
```

- 定义延迟回调的签名：接收 `&mut T`（目标应用/组件）、`&mut Cx`（图形上下文）、`&mut Scope`（脚本作用域）。
- 通过 blanket implementation 自动为所有满足签名的闭包实现。
- 要求 `Send + 'static` 以支持跨线程投递。
- `FnOnce` 意味着闭包只能执行一次，这与 `Mutex<Option<...>>` 的 `take()` 设计一致。

---

## 执行流程总结

```
线程 A (后台)                  线程 B (UI 主线程)
     |                              |
     |  UiRunner::defer(f)          |
     |    → 创建 UiRunnerAction     |
     |    → Cx::post_action(action) |
     |    → 发送到通道              |
     |    → SignalToUI::set_action  |
     |                              |  ← 信号唤醒事件循环
     |                              |  Cx::handle_action_receiver()
     |                              |    → 从通道接收动作
     |                              |    → 存入 new_actions
     |                              |  App::handle_event()
     |                              |    → UiRunner::handle()
     |                              |    → 匹配 key、取闭包、执行
     |                              |
     |  block_on 场景:              |
     |  rx.recv() ←—————— tx.send(result)
```

---

## 完整 Test / 使用示例

```rust
// 定义应用
struct MyApp { ... }

impl AppMain for MyApp {
    fn handle_event(&mut self, cx: &mut Cx, event: &Event) {
        // 处理 UiRunner 动作
        self.ui_runner().handle(cx, event, &mut Scope::empty(), self);
        // ... 其他事件处理
    }
}

// 在后台线程使用
fn background_work(runner: UiRunner<MyApp>) {
    // 非阻塞调度
    runner.defer(|app, cx, scope| {
        app.update_something(cx);
    });

    // 阻塞等待结果
    let result = runner.block_on(|app, cx, scope| {
        app.compute_value(cx)
    });
}
```
