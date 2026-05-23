# `lib.rs` — Splash 标准库入口 / 泵循环调度

## 文件位置
`platform/script/std/src/lib.rs`

---

## 总体职责
这个文件是 `makepad_script_std` crate 的根模块。它 re-export 了 `makepad_network` 和 `makepad_script` 两个核心依赖，声明了 `data/fs/net/run/task/vm` 六个子模块，并提供了**标准库注册**和**事件泵（pump）循环**的顶层调度函数。

---

## 关键函数

### `pub fn script_mod(vm: &mut ScriptVm)`
- 将 `fs`、`run`、`task`、`net` 这四个子模块中的脚本 API 注册到指定的 `ScriptVm` 实例上。
- 调用顺序为 `fs` → `run` → `task` → `net`，其中 `task` 必须在 `net` 之前注册以确保任务调度所需的基础设施就绪。
- `data` 和 `vm` 子模块不提供独立的 `script_mod` 函数，`data` 定义数据结构体，`vm` 提供纯 Rust 工具函数。

### `pub fn pump<H: Any>(host: &mut H, std: &mut ScriptStd, script_vm: &mut Option<Box<ScriptVmBase>>)`
- 一次性执行所有标准库的异步事件泵操作，包括子进程输出处理、Socket 流数据处理、HTTP 服务器请求处理、以及任务协程恢复。
- 调用顺序为：子进程 → Socket 流 → HTTP 服务器 → 任务调度。这个顺序确保任务调度在其他 I/O 事件处理之后运行，从而使 I/O 触发的协程恢复能够立即执行。
- 参数 `host` 是宿主应用的自定义状态，通过泛型 `H: Any` 透传给子模块的泵函数。

### `pub fn pump_network_runtime<H: Any>(...) -> Vec<makepad_network::NetworkResponse>`
- 专门用于排放网络运行时（`NetworkRuntime`）中的待处理事件队列。
- 调用 `net::drain_network_runtime` 从运行时中收集所有 `NetworkResponse`，如果非空则继续调用 `net::handle_script_network_events` 分发 HTTP/WebSocket 回调，然后运行任务调度。
- 返回收集到的响应列表，供调用方进一步处理。这个函数的设计允许宿主在自定义的泵循环中单独管理网络运行时，而不必依赖完整的 `pump`。

### `pub use` 重导出
- `pub use makepad_network;` 和 `pub use makepad_script;` 使得调用方在 `Cargo.toml` 中仅依赖此 crate 就能访问两个底层 crate。
- 通过 `pub use data::*;` / `net::*` / `run::*` / `task::*` / `vm::*` 将各子模块的公共类型和函数提升到 crate 根级，简化调用方的导入。
- `fs` 模块的内容没有 `pub use` 重导出，因为其函数（`read`/`write` 等）仅在脚本 DSL 中使用，不会在 Rust 侧直接调用。
