# `task.rs` — 协程任务与 Promise 机制

## 文件位置
`platform/script/std/src/task.rs`

---

## 总体职责
这个文件实现了 Splash 脚本语言的核心协程调度系统——`std.task()` 和 `std.promise()`。它基于一个共享的 `Rc<RefCell<Vec<ScriptTask>>>` 任务列表，通过生产者-消费者模式的暂停/恢复机制调度协程。文件提供了从 Rust 侧添加钩子和触发线程恢复的 API，供网络、子进程等标准库模块在 I/O 事件到达时唤醒等待的协程。

---

## 关键数据结构

### `ScriptTask`
- 表示一个运行中的协程任务或 promise：
  - `start_task: Option<ScriptFnRef>` —— 启动函数引用（对于 `task` 是可选的，对于 `promise` 是必需的构造器）。
  - `handle: ScriptHandle` —— 脚本侧的 GC 句柄。
  - `queue: ScriptArrayRef` —— 任务的消息队列（供 send/recv 操作使用）。
  - `max_depth: usize` —— 队列最大深度控制背压；0 表示无限制。
  - `ended: bool` —— 标记是否已结束（end/resolve 已被调用）。
  - `send_pause: VecDeque<ScriptThreadId>` —— 因队列满而暂停的发送方协程 ID 列表。
  - `recv_pause: VecDeque<ScriptThreadId>` —— 因队列空而暂停的接收方协程 ID 列表。

### `ScriptTasks`
- `tasks: Rc<RefCell<Vec<ScriptTask>>>` —— 所有活跃任务的可共享、可修改列表。
- `pending_resumes: VecDeque<ScriptThreadId>` —— 等待恢复的协程线程 ID 队列（由 I/O 泵函数通过 `queue_script_thread_resume` 填充）。
- `hooks: ScriptTaskHooks` —— 钩子集合（线程完成钩子和泵钩子）。

### `ScriptTaskHooks` / `ScriptTaskGc`
- `ScriptTaskHooks`：`on_thread_completed: Vec<ScriptTaskOnThreadCompletedHook>` 和 `pump: Vec<ScriptTaskPumpHook>`。这些钩子允许宿主代码在协程完成时或任务泵循环中执行自定义逻辑（如 Makepad Studio 的日志记录）。
- `ScriptTaskGc` 实现 `ScriptHandleGc`，在 GC 回收句柄时从任务列表中移除对应的 `ScriptTask`。

---

## 关键函数

### `pub fn add_script_task_on_thread_completed_hook(std, hook)`
- 向 `std.data.tasks.hooks.on_thread_completed` 列表中添加一个线程完成钩子。钩子函数的签名为 `fn(&mut dyn Any, ScriptThreadId, ScriptValue) -> bool`，返回值表示是否消费了该结果。
- 通过函数指针比较去重，避免重复添加相同的钩子。

### `pub fn add_script_task_pump_hook(std, hook)`
- 类似地添加泵钩子（`fn(&mut dyn Any) -> bool`，返回值表示是否有进展）。泵钩子在每次任务调度循环中被调用。

### `pub fn queue_script_thread_resume(std, thread_id)`
- 将一个线程 ID 放入 `pending_resumes` 队列，供下次 `handle_script_tasks` 调度时恢复。
- 这是网络、子进程等 I/O 模块在事件到达时唤醒协程的标准途径。

### `pub fn handle_script_tasks(host, std, script_vm)`
- 这是协程调度的主循环，在一个 `loop { ... if !progressed { break; } }` 中反复执行直到没有进展。
- 调度优先级：
  1. **`pending_resumes` 中的线程**：优先处理被外部 I/O 事件触发的恢复请求。
  2. **启动新任务**：如果某个 `ScriptTask` 的 `start_task` 尚未被消费，立即调用它（传入该任务的 handle 作为参数）。
  3. **接收方恢复**：如果某个任务的 `recv_pause` 非空且队列中有数据，恢复一个接收方线程。
  4. **发送方恢复**：如果某个任务的 `send_pause` 非空且队列长度小于 `max_depth`，恢复一个发送方线程。
- 对于线程恢复：调用 `vm::with_vm_thread` 在指定线程上执行 `vm.resume()`。如果恢复后线程不再处于暂停状态（即线程结束），运行 `run_script_task_thread_completed_hooks` 钩子。
- 在所有调度步骤之后运行 `run_script_task_pump_hooks`，如果任何一个钩子返回 `true`，标记为 `progressed`。

---

## 脚本模块注册

### `pub fn script_mod(vm: &mut ScriptVm)`
- 创建两种句柄类型：`task_type`（`id_lut!(task)`）和 `promise_type`（`id_lut!(promise)`）。

**内部辅助函数：**

- `add_send_method(vm, handle_type, fn_id, end_on_send)`：
  - 向指定句柄类型添加一个"发送"方法（对于 task 是 `emit`/`end`，对于 promise 是 `resolve`）。
  - 参数提取：如果调用时无参数，发送 `NIL`；一个参数发送该参数；多个参数发送整个 `args` 数组。
  - 将值 push 到任务的 `queue` 数组中。如果 `end_on_send` 为 true（`end`/`resolve`），标记 `task.ended = true`。
  - 如果队列满（`array_len >= task.max_depth`），暂停当前线程并放入 `send_pause` 队列（上限 100 个暂停）。

- `add_recv_method(vm, handle_type, fn_id, wait_for_end)`：
  - 向指定句柄类型添加一个"接收"方法（对于 task 是 `next`/`last`，对于 promise 是 `await`）。
  - 尝试从队列头部 pop 一个值。如果成功且不等待结束，直接返回该值。
  - 如果任务已结束（`ended`），返回 `NIL`。
  - 如果队列为空且未结束，暂停当前线程并放入 `recv_pause` 队列（上限 100 个暂停）。

- `add_queue_getter(vm, handle_type)`：
  - 为句柄类型添加 `queue` 属性的读取器，返回该任务的底层数组引用。

- `create_task_handle(vm, handle_type, start_task, max_depth) -> ScriptValue`：
  - 创建一个新的 `ScriptTaskGc` 句柄对象，注册到 VM 堆中。
  - 创建一个新数组和 `ScriptArrayRef`，构造 `ScriptTask` 并 push 到共享的任务列表中。
  - 返回句柄作为脚本值。

**注册的方法：**

| 模块路径 | 方法 | 类型 | 说明 |
|---------|------|------|------|
| `std.task(start_fn_or_depth)` | — | 构造函数 | 如果参数为函数，创建带启动函数的 task（max_depth=1）；如果为数字，创建有最大队列深度的 task（无启动函数） |
| `std.promise(start_fn)` | — | 构造函数 | 创建 promise，max_depth=1，start_fn 可选 |
| `task.emit(...)` | send | 发送值到队列（不结束） |
| `task.end(...)` | send+结束 | 发送最后一个值并标记 ended |
| `task.next()` | recv | 从队列取下一个值（不等待结束） |
| `task.last()` | recv+等待结束 | 从队列取值，等待任务结束 |
| `promise.resolve(...)` | send+结束 | 解析 promise（等同于 end） |
| `promise.await()` | recv+等待结束 | 等待 promise 解析（等同于 last） |

两个类型都暴露 `queue` 属性供脚本侧检查队列内容。
