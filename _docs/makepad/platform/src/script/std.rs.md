# `platform/src/script/std.rs` — 脚本标准库集成桥

## 文件职责

该文件在 Makepad 的 `Cx`（平台上下文）上添加了一系列桥接方法，将 `Cx` 与 `makepad_script_std` 标准库连接起来。它实现了 `Cx↔ScriptVm` 之间的状态交换模式，使得平台层可以安全地调用脚本引擎，同时脚本引擎也可以访问平台层的功能和状态。

---

## 类型别名

```rust
pub use makepad_script_std::{fs, net, run};
pub type CxScriptTaskOnThreadCompletedHook = ScriptTaskOnThreadCompletedHook;
pub type CxScriptTaskPumpHook = ScriptTaskPumpHook;
```

- `fs`、`net`、`run` 模块从 `makepad_script_std` 重导出为平台层公共 API
- 两个 hook 类型别名使得 `Cx` 上的方法签名无需直接依赖 `makepad_script_std` 的类型路径

---

## 核心方法

### `cx.script_std()` / `cx.script_std_mut()`

对 `Cx::script_data.std` 的快捷访问器，分别返回 `&ScriptStd` 和 `&mut ScriptStd`。

### `cx.with_script_std_vm(f)`

这是一个**内部低级工具方法**，模式如下：
1. 获取 `self` 和 `self.script_data.std` 的原始指针
2. 获取 `self.script_vm` 的原始指针
3. 通过 `unsafe` 块将三个指针解引用后传递给闭包 `f`

使用原始指针而非引用的原因是 Rust 的借用检查器无法证明从同一个 `&mut self` 同时借用 `script_data.std` 和 `script_vm` 是安全的（它们虽然是同一结构体的不同字段，但在嵌套结构体上借用规则受限）。通过原始指针绕过借用检查器，但调用方（`with_vm` 系列方法）保证了实际的借用安全性。

### `cx.with_vm_and_async(f)`

通过 `with_script_std_vm` 桥接调用 `makepad_script_std::with_vm_and_async`。这是最常用的入口——它：
1. 从 `Cx` 获取保存的 `ScriptVmBase`（如果有）
2. 在 `ScriptStd` 上获取 `Box<ScriptVmBase>`，构造临时的 `ScriptVm` 包装
3. 执行闭包 `f`
4. 处理异步完成事件（通过 `ScriptTaskOnThreadCompletedHook`）
5. 将 `ScriptVmBase` 放回到 `Cx` 或 `ScriptStd` 中
6. 返回结果

这是需要**创建新脚本线程**或执行 `await` 相关操作时的首选方法。

### `cx.with_vm(f)`

调用 `makepad_script_std::with_vm`，执行一个共享 `ScriptVm` 上的同步操作。不创建新线程，不会泵送异步完成事件。适用于只需要快速访问 VM 状态的场景。

### `cx.with_vm_thread(thread_id, f)`

调用 `makepad_script_std::with_vm_thread`，在指定线程的 `ScriptVm` 上执行操作。用于向特定脚本线程发送消息或触发特定线程上的回调执行。

### `cx.eval(script_mod)`

调用 `makepad_script_std::eval`，在共享 VM 上评估一个 `ScriptMod`（脚本模块）。返回评估结果值。这是平台层执行脚本代码的主要入口。

### `cx.add_script_task_on_thread_completed_hook(hook)`

注册一个钩子，当脚本线程完成异步任务时触发。用于任务调度器的完成通知机制。

### `cx.add_script_task_pump_hook(hook)`

注册一个泵送钩子，在主循环的每次 pump 中调用，用于驱动异步脚本任务的进度。

### `cx.queue_script_thread_resume(thread_id)`

将一个脚本线程标记为待恢复状态，在下一次 pump 时该线程的等待操作将被继续执行。

### `cx.set_script_task_trace(enabled)`

启用/禁用脚本任务跟踪，用于调试异步任务执行流程。

### `cx.handle_script_tasks()`

通过 `with_script_std_vm` 调用 `makepad_script_std::handle_script_tasks`。通常在每一帧末尾调用，处理已完成的异步脚本任务，将它们的结果传递回脚本线程。

### `cx.handle_script_signals()`

通过 `with_script_std_vm` 调用 `makepad_script_std::pump`。驱动脚本后端的信号处理——检查异步 I/O 状态、推进网络请求进度等。

### `cx.handle_script_web_socket_event(event)`

将 WebSocket 网络事件传递到脚本后端。通过 `makepad_script_std::handle_script_web_socket_event` 处理，使脚本可以接收 WebSocket 消息。

### `cx.handle_script_network_events(responses)`

这是最复杂的方法。它遍历所有 `NetworkResponse`：

1. **HTTP 资源检查**：对于每个 HTTP 响应（`HttpResponse`、`HttpError`、`HttpStreamChunk`、`HttpStreamComplete`、`HttpProgress`），先检查 `request_id` 是否属于脚本资源加载系统（`is_http_resource`）

2. **资源响应处理**：如果是资源 HTTP 请求：
   - **`HttpResponse`**：检查状态码是否为 2xx。如果是，将响应体存入对应资源；否则记录错误日志并标记为 `Error`。调用 `self.redraw_all()` 触发界面重绘。
   - **`HttpError`**：记录错误信息到资源中
   - 其他 HTTP 响应类型忽略

3. **日志记录**：对于每个匹配的 HTTP 资源，构建 `resource_info` 字符串包含绝对路径、web_url 和依赖路径，便于调试

4. **标准网络事件处理**：无论响应是否属于资源系统，最后都调用 `makepad_script_std::handle_script_network_events` 将事件传递到脚本标准库的网络处理层（供 `std::net::fetch` 等脚本 API 使用）

#### 设计要点

- `Cx` 的网络事件处理被分为两层：**平台层资源加载**和**脚本层网络 API**
- 资源加载的 HTTP 响应直接处理并填充数据，不经过脚本线程
- 其他网络事件（包括 WebSocket）通过 `ScriptStd` 路由到等待的脚本线程
- `self.redraw_all()` 在资源加载完成后触发界面重绘，确保新加载的纹理/资源能立即显示
