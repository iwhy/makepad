# wayland_app.rs — Wayland Application Event Loop

**文件路径**: platform/src/os/linux/wayland/wayland_app.rs (122行)
**核心用途**: 封装 Wayland 事件循环的高层逻辑，管理连接、事件队列和定时器分发。

## Types/Structs

### `WaylandApp`
Wayland 应用的主事件循环。

| 字段 | 类型 | 描述 |
|------|------|------|
| `connection` | `Connection` | Wayland 连接 |
| `event_queue` | `EventQueue<WaylandState>` | Wayland 事件队列，与 WaylandState 关联 |
| `state` | `WaylandState` | Wayland 协议状态 |
| `event_callback` | `Option<Box<dyn FnMut(&mut WaylandApp, XlibEvent) -> EventFlow>>` | 事件回调函数 |

## Key Methods

### `new(connection, event_queue, state, event_callback) -> Self`
构造 WaylandApp。参数：Wayland 连接、事件队列、初始状态、事件回调。

### `event_loop()`
主事件循环：
1. 发送初始 `XlibEvent::Paint`
2. 循环处理 `state.event_flow`：
   - **EventFlow::Exit**: 退出
   - **EventFlow::Wait**: 更新定时器，处理键盘重复定时器，准备读取事件队列（`prepare_read` + `select`）
   - **EventFlow::Poll**: 更新定时器，调用 `event_loop_poll`

### `event_loop_poll()`
单次事件轮询：
1. `self.event_queue.flush()` — 刷新待发送的协议请求
2. `prepare_read()` — 若可，读取并调度待处理事件
3. 发送 `XlibEvent::Paint` 回调

### `do_callback(event)`
取出 `event_callback`，调用回调函数，处理 `EventFlow::Exit` 退出。

### `terminate_event_loop()`
设置 `event_loop_running = false`。

### `start_timer(id, timeout, repeats)` / `stop_timer(id)` / `time_now()`
定时器方法的委托，转发给 `WaylandState` 的 `SelectTimers`。

## Implementation Details
- 错误处理：flush/read/dispatch 失败时记录警告并终止事件循环
- 定时器通过 POSIX `select()` 系统调用实现等待
- 键盘重复定时器在 `WaylandState::handle_key_repeat_timer` 中处理
