# widget_async.rs — ScriptAsync 异步调用与跨线程 UI Handle

## 文件概述

实现了 Makepad 的异步任务执行机制。核心组件是 `ScriptAsync` 和 `UiHandle`/`UiPtrHandle`，允许在主渲染线程之外执行耗时操作（网络请求、文件 I/O、计算密集型任务），并将结果安全地发送回 UI 线程。

---

## `ScriptAsync` 结构体

```rust
pub struct ScriptAsync<T: 'static + Send> {
    handle: Option<JoinHandle<()>>,      // tokio 任务句柄
    sender: UnboundedSender<ScriptAsyncMessage<T>>,  // 结果发送端
    receiver: UnboundedReceiver<ScriptAsyncMessage<T>>, // 结果接收端
}
```

核心设计模式：**通道桥接** — `ScriptAsync` 在创建时立即生成一个 unbounded 通道，发送端可在线程间移动（`Send`），接收端在主线程持有。

### 方法

```rust
impl<T: 'static + Send> ScriptAsync<T> {
    pub fn new() -> Self
    pub fn spawn<F>(&mut self, fut: F) where F: Future<Output = T> + Send + 'static, T: 'static
    pub fn spawn_blocking<F>(&mut self, f: F) where F: FnOnce() -> T + Send + 'static, T: 'static
    pub fn is_ready(&self) -> bool
    pub fn take(&mut self) -> Option<T>
}
```

- `new()`：创建新的异步通道。与 `ScriptVm` 绑定，`cx.async_scope()` 提供 vm 关联的通道。
- `spawn(fut)`：在 tokio 运行时上 spawn 一个异步任务。任务返回 `T` 后通过通道发送回主线程。
- `spawn_blocking(f)`：在 tokio 的 blocking pool 上执行 CPU 密集型任务，不阻塞异步运行时。
- `is_ready()`：非阻塞检查通道中是否有可用结果。
- `take()`：取出结果，返回 `Option<T>`。在 `match_frame_event` 或 `handle_event` 中调用。

### 使用流程

```rust
// 1. 创建异步处理器
let mut async_fetch = ScriptAsync::new();

// 2. 在 UI 线程之外执行任务
async_fetch.spawn(async {
    http_client::get("https://api.example.com/data").await
});

// 3. 在主循环中检查结果
fn handle_event(&mut self, cx: &mut Cx, event: &Event) {
    self.match_event(cx, event);
    if let Some(result) = self.async_fetch.take() {
        // 更新 UI
    }
}
```

---

## `UiHandle<T>` 和 `UiPtrHandle<T>`

为跨线程 UI 操作提供句柄：

### `UiHandle<T>`

```rust
pub struct UiHandle<T: 'static + Send> {
    sender: UnboundedSender<T>,
}
```

- `Clone` + `Send`：可以安全地发送到其他线程。
- `send(val: T)`：向主线程发送一个值。
- `downgrade()`：创建 `WeakUiHandle`（防止循环引用导致泄漏）。

### `UiPtrHandle<T>`

类似 `UiHandle`，但接收引用类型而非值类型。通过 `UiHandle` + 额外映射实现：

```rust
pub struct UiPtrHandle<T: 'static + Send + Clone> {
    inner: UiHandle<T>,
}
```

### 获取方式

```rust
// 从 Cx 获取
let handle = cx.ui_handle::<MyMessage>();

// 从 UiHandle 构造指针版本（需 Clone）
let ptr_handle = UiPtrHandle::new(my_handle);
```

---

## ScriptAsync 宏

```rust
#[derive(ScriptAsync)]
pub struct MyAsyncComponent {
    // 自动生成 async_scope 相关方法
}
```

该派生宏自动为结构体生成 `async_scope` 相关的辅助代码，使其能方便地与 `ScriptVm` 的异步系统集成。

---

## 设计要点

1. **线程安全**：所有跨线程数据必须实现 `Send + 'static`，编译器强制执行。
2. **非阻塞检查**：`is_ready()` / `take()` 都在主线程调用，不会阻塞渲染。
3. **unbounded 通道**：使用 `async_channel::unbounded`，避免背压导致的死锁。消息积压理论上无上限，但在实际使用中 `take()` 在每一帧都会被调用。
4. **生命周期**：`ScriptAsync` 必须与拥有它的 Widget 生命周期一致，确保在 Widget 销毁前处理所有挂起的消息。
5. **tokio 集成**：底层依赖 tokio 运行时，需确保应用已经初始化 tokio（`cargo-makepad` 或用户代码负责）。
