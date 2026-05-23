# `vm.rs` — ScriptVm 生命周期管理与扩展 trait

## 文件位置
`platform/script/std/src/vm.rs`

---

## 总体职责
这个文件定义了如何从 Rust 宿主代码中安全地借用和操作 `ScriptVm` 实例。核心在于 `with_vm` 系列函数——它们从 `Option<Box<ScriptVmBase>>` 中取出虚拟机的内部状态，闭包执行完毕后再归还。文件还提供了用于直接从标准库状态中访问指定类型的类型擦除扩展。

---

## 关键 Trait

### `pub trait ScriptVmStdExt`
- 对 `ScriptVm<'a>` 和 `&mut dyn Any` 实现了 `std_ref<T>` 与 `std_mut<T>` 方法。
- 内部直接调用 `self.std.downcast_ref::<T>()` 或 `downcast_mut::<T>()`，将类型擦除的 `std` 字段向下转型为具体的 `ScriptStd` 类型。
- 这是标准库模块（如 `net.rs`、`task.rs`、`run.rs`）访问 `ScriptStd` 数据的标准途径，例如 `vm.std_mut::<ScriptStd>()`。

---

## 关键函数

### `pub fn with_vm_and_async<H, F, R>(host, std, script_vm, f) -> R`
- 从 `script_vm` 中 `take()` 出 `Box<ScriptVmBase>`，若为 `None` 则 panic（"Script VM swapped off"）。
- 调用前先设置线程到第一个未暂停的线程（`set_current_to_first_unpaused_thread`）。
- 构造 `ScriptVm { host, std, bx }`，传入闭包 `f` 执行。闭包返回后，将 `bx` 归还到 `script_vm`。
- 最后调用 `task::handle_script_tasks` 处理闭包执行过程中产生的任务恢复请求。
- 与 `with_vm` 的区别在于：**不调用 `drain_errors`**，允许异步场景下错误被保留到后续的泵循环中处理。

### `pub fn with_vm<H, F, R>(host, std, script_vm, f) -> R`
- 逻辑与 `with_vm_and_async` 相同，但在闭包执行完毕后调用 `vm.drain_errors()` 立即处理 VM 中的待处理错误。
- 适用于同步执行场景（如直接调用 `eval` 或同步回调），确保错误不会被延迟。

### `pub fn with_vm_thread<H, F, R>(host, std, script_vm, thread_id, f) -> R`
- 类似 `with_vm`，但不是使用第一个未暂停线程，而是通过 `thread_id` 明确指定要激活的协程线程。
- 调用 `bx.threads.set_current_thread_id(thread_id)` 设置当前线程上下文。
- 用于在任务调度器中恢复特定协程的执行，如 `handle_script_tasks` 中对暂停线程的 `resume` 调用。

### `pub fn eval<H: Any>(host, std, script_vm, script_mod) -> ScriptValue`
- 基于 `with_vm_and_async` 的便捷包装，直接调用 `vm.eval(script_mod)` 执行一段脚本模块。
- 返回脚本执行结果值，适用于需要从 Rust 侧触发 Splash 代码执行的场景。
