# `run.rs` — 子进程管理标准库

## 文件位置
`platform/script/std/src/run.rs`

---

## 总体职责
这个文件实现了 Splash 脚本中的子进程启动和输出流监控功能。它通过 `std::process::Command` 启动外部进程，使用三个独立的后台线程分别处理 stdout、stderr 和 stdin 流，通过 `FromUISender`/`ToUIReceiver` 通道将进程输出事件传递到 UI 线程并派发给脚本回调。

---

## 关键数据结构

### `ChildProcess`（私有）
- 内部结构，包装了 `Child` 进程句柄以及两个通信通道：
  - `in_send: FromUISender<ChildIn>` —— 用于向后台线程发送 stdin 数据或终止信号。
  - `out_recv: ToUIReceiver<ChildOut>` —— 用于接收来自后台线程的 stdout/stderr/term 事件。
- 不暴露给脚本侧或 Rust 外部。

### `ScriptChildProcessState`
- 公开结构体，包含：
  - `id: LiveId` —— 该子进程的唯一标识符（用于 GC 或后续识别）。
  - `child: ChildProcess` —— 内部子进程状态。
  - `events: ScriptChildEvents` —— 脚本回调函数容器。

### `ScriptChildEvents`
- `#[derive(Script, ScriptHook)]`，注册为脚本类型。
- 可选回调：`on_stdout`、`on_stderr`、`on_term`（全部为 `Option<ScriptFnRef>`）。

### `ScriptChildCmd`
- `#[derive(Script, ScriptHook)]`，注册为脚本类型。
- 字段：`cmd: String`（可执行路径）、`args: Option<Vec<String>>`、`env: Option<HashMap<String, String>>`（环境变量）、`cwd: Option<String>`（工作目录）。

---

## 关键函数

### `pub fn script_mod(vm: &mut ScriptVm)`
- 创建 `mod.run` 模块，注册 `ScriptChildEvents` 和 `ScriptChildCmd` 类型。
- 注册 `run.child(cmd, events)` 方法。

### `run.child(cmd, events) -> LiveId`
- 接收两个脚本对象参数：`ScriptChildCmd`（命令配置）和 `ScriptChildEvents`（事件回调）。
- 通过 `script_has_proto!` 检查两个参数的类型正确性。
- 使用 `ScriptChildCmd` 的字段构造 `std::process::Command`：
  - 设置环境变量（遍历 `HashMap`）。
  - 设置命令行参数（`cmd_build.args(args)`）。
  - 设置管道标准流（`stdin(Stdio::piped())` / `stdout(Stdio::piped())` / `stderr(Stdio::piped())`）。
  - 设置工作目录（如果指定）。
- 调用 `ChildProcess::spawn(cmd_build)` 启动进程和 IO 线程。
- 成功时生成唯一的 `LiveId`，将 `ScriptChildProcessState` 加入 `std.data.child_processes` 列表，并返回 `id.escape()` 给脚本侧。
- 失败时抛出 "child process error"。

### `impl ChildProcess::spawn(command) -> Result<ChildProcess, std::io::Error>`
- 调用 `command.spawn()` 启动操作系统进程。提取子进程的 stdin/stdout/stderr。
- 创建三个后台线程：

  **stdout 读取线程：**
  - 使用 `BufReader` 逐行读取子进程的 stdout。
  - 每读取一行，通过 `out_send.send(ChildOut::StdOut(line))` 发送到 UI 线程。
  - 当读取返回长度为 0（EOF）时，发送 `ChildOut::Term` 终止信号，然后发送 `ChildIn::Term` 通知 stdin 线程退出。

  **stderr 读取线程：**
  - 与 stdout 线程类似，但发送 `ChildOut::StdErr(line)` 事件。
  - 读取到 EOF 时静默退出，不发送 Term 信号（由 stdout 线程负责发送 Term）。

  **stdin 写入线程：**
  - 循环接收 `in_recv` 通道中的 `ChildIn::Send(line)` 或 `ChildIn::Term` 消息。
  - 当收到 Send 时，将字符串写入子进程的 stdin 并 flush。
  - 当收到 Term 时，退出循环，结束线程。

- 返回 `ChildProcess` 结构体，UI 线程侧保留 `in_send` 和 `out_recv` 通道。

### `pub fn handle_script_child_processes(host, std, script_vm)`
- 逐 `while` 循环遍历 `std.data.child_processes` 列表（手动索引管理以实现并发安全的移除）。
- 对每个进程，从 `out_recv` 通道 try_recv 所有可用事件：
  - `ChildOut::StdOut(s)` —— 通过 `with_vm_and_async` 调用 `on_stdout(s)` 回调。
  - `ChildOut::StdErr(s)` —— 调用 `on_stderr(s)` 回调。
  - `ChildOut::Term` —— 调用 `on_term()` 回调，设置 `term = true` 标记，跳出内层循环。
- 如果 `term` 为 true（子进程已终止），从列表中 `remove(i)` 该进程状态；否则递增索引继续检查下一个。
- 手动索引管理：当进程终止并移除时，不递增 `i`，因为 `remove` 会将后续元素前移。
