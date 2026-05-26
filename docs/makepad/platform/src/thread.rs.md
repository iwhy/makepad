# `thread.rs` — 工作线程池抽象

## 用途

`thread.rs` 实现了三种不同策略的工作线程池——`RevThreadPool`（无标签任务池）、`TagThreadPool`（带标签的任务池，支持替换同名任务）、`MessageThreadPool`（带消息递送的线程池）。它们共同为 Makepad 引擎提供了灵活的后台计算能力，适用于离线编译、资源加载、网络请求等非 UI 线程的工作。

## RevThreadPool

### 结构
```rust
pub struct RevThreadPool {
    tasks: Arc<Mutex<Vec<Box<dyn FnOnce() + Send + 'static>>>>,
}
```
一个简单的工作窃取风格线程池，所有线程共享一个任务队列。

### `new(cx: &mut Cx, num_threads: usize)`

创建指定数量的工作线程。每个线程进入一个无限循环：从 `tasks` 队列中 `pop` 获取任务并执行，若无任务则忙等待（通过 `tasks.lock()` 的阻塞行为）直到有新任务到达。使用 `cx.spawn_thread` 来创建线程以确保与事件循环的集成。

### `execute<F>(&self, task: F)`

将任务插入到队列**头部**（`insert(0, ...)`），因此下一个空闲线程会优先执行这个最新添加的任务。这种行为是"后进先出"的——最近提交的任务优先执行。

### `execute_rev<F>(&self, task: F)`

将任务追加到队列**尾部**（`push`），因此任务按提交顺序依次执行。相比 `execute` 的 LIFO 策略，这是标准的 FIFO 策略。

## TagThreadPool

### 结构
```rust
pub struct TagThreadPool<T: Clone + Send + 'static + PartialEq> {
    tasks: Arc<Mutex<Vec<(T, Box<dyn FnOnce(T) + Send + 'static>)>>>,
}
```
带标签的线程池，每个任务关联一个 `T` 类型的标签。核心特性：**提交同名标签的新任务时自动移除旧任务**。

### `new(cx: &mut Cx, num_threads: usize)`

与 `RevThreadPool::new` 类似的初始化，但线程在队列为空时**睡眠 50ms**而不是忙等待。这是因为带标签的任务通常频率较低，使用睡眠可以降低 CPU 占用。

### `execute<F>(&self, tag: T, task: F)`

提交一个带标签的任务，插入队列头部。在插入前先使用 `retain(|v| v.0 != tag)` 移除所有同标签的已有任务。这意味着对同一个标签连续调用 `execute`，只有最新的任务会执行——之前的都会被取消。`task` 闭包接收 `T`（标签）作为参数。

### `execute_rev<F>(&self, tag: T, task: F)`

与 `execute` 逻辑相同（去重后追加到尾部），区别在于插入位置是队列末尾而非头部。

## MessageThreadPool

### 结构
```rust
pub struct MessageThreadPool<T: Clone + Send + 'static> {
    sender: Sender<Box<dyn FnOnce(Option<T>) + Send + 'static>>,
    msg_senders: Vec<Sender<T>>,
}
```

最复杂的线程池，每个工作线程除了接收任务外，还有一个专用的消息通道。

### `new(cx: &mut Cx, num_threads: usize)`

创建 `n` 个工作线程，每个线程除共享的任务通道外，还拥有独立的 `msg_recv` 通道。线程在主循环中：
1. 从共享通道接收一个任务（阻塞等待）。
2. 尝试从专用通道 `try_recv` 获取所有待处理消息（使用 `while let Ok(msg) = msg_recv.try_recv()` —— 只保留最后一条消息）。
3. 将累积的消息（`Option<T>`）传给任务闭包执行。

### `send_msg(&self, msg: T)`

广播一条消息到所有工作线程的专用通道。每个线程独立接收并保留该消息的最新值。

### `execute<F>(&self, task: F)`

通过共享通道发送一个任务到线程池。任务将被任意一个空闲线程拾取。任务闭包接收 `Option<T>`（该线程最近收到的消息）。

## 三种线程池对比

| 特性 | `RevThreadPool` | `TagThreadPool` | `MessageThreadPool` |
| --- | --- | --- | --- |
| 去重语义 | 无 | 标签去重 | 无 |
| 空闲行为 | 阻塞 | 睡眠 50ms | 阻塞 |
| 任务参数 | 无 | 标签值 | 可选消息 |
| 消息广播 | 无 | 无 | **有**（专用通道） |
| 典型场景 | 编译/加载 | 资源重新编译 | 带状态更新的计算 |

## 设计要点

1. **`RevThreadPool` 的类栈调度**：`execute` 插入队列头部的行为使线程池呈现 LIFO 调度策略，这通常有利于缓存局部性——最近提交的任务更可能仍在 CPU 缓存中。

2. **`TagThreadPool` 的懒取消**：通过标签匹配自动移除旧任务，避免执行过期的工作。这在 UI 框架中特别有用——例如当用户快速调整参数时，只执行最新的重编译任务。

3. **`MessageThreadPool` 的消息嗅探**：每个线程在任务执行前检查其专用消息通道，只保留最新消息。这实现了类似"带状态的工作线程"模式——线程可以在执行任务时感知到最新状态而无需额外同步。

4. **与事件循环集成**：所有线程池都接收 `&mut Cx` 并使用 `cx.spawn_thread` 创建线程，确保线程与 Makepad 事件循环的生命周期管理一致。

5. **双通道解耦**：`MessageThreadPool` 将任务流与消息流分离——任务通道是共享的（竞争消费），消息通道是独占的（广播通知），避免了任务和状态更新的耦合。
