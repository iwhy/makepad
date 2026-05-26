# `data.rs` — 标准库运行时状态结构体

## 文件位置
`platform/script/std/src/data.rs`

---

## 总体职责
这个文件定义了 Splash 标准库在运行时维护的全部可变状态。`ScriptStd` 作为顶层存储容器，持有网络运行时引用以及所有正在进行的异步操作的状态（子进程、WebSocket、Socket 流、HTTP 请求、HTTP 服务器、任务协程）。

---

## 关键结构体

### `pub struct ScriptStd`
- 标准库运行时状态的总入口。默认构建时 `net` 为 `None`（无网络能力）。
- `pub net: Option<Arc<NetworkRuntime>>`：网络运行时的可选引用。当需要 HTTP/WebSocket/Socket 功能时，必须通过 `with_network_runtime` 或 `set_network_runtime` 设置。
- `pub data: ScriptData`：所有异步操作状态的聚合容器。

**构造函数：**
- `ScriptStd::new()` —— 创建无网络能力的空标准库实例，`net` 为 `None`。
- `ScriptStd::with_network_runtime(net)` —— 创建一个绑定了网络运行时的实例，适用于需要网络 I/O 的测试和生产场景。
- `set_network_runtime(&mut self, net)` —— 在运行时动态注入网络运行时引用，用于延迟初始化。

### `pub struct ScriptData`
- 使用 `#[derive(Default)]`，所有字段在创建时均为空。
- `pub tasks: ScriptTasks` —— 管理所有 Splash 协程任务（`std.task()` 和 `std.promise()` 创建的异步任务）。通过 `Rc<RefCell<Vec<ScriptTask>>>` 存储。
- `pub child_processes: Vec<ScriptChildProcessState>` —— 所有正在运行的子进程状态列表。每个条目包含进程的 stdout/stderr 通道和事件回调。
- `pub web_sockets: Vec<ScriptWebSocket>` —— 客户端 WebSocket 连接列表。每个连接包含 socket_id 和事件回调（on_opened/on_closed/on_binary/on_string/on_error）。
- `pub socket_streams: Rc<RefCell<Vec<ScriptSocketStream>>>` —— TCP/TLS Socket 流列表。使用 `Rc<RefCell<>>` 包装以允许在多处共享可变访问（GC 句柄也需要引用同一份数据）。
- `pub http_requests: Vec<ScriptHttp>` —— 正在进行的 HTTP 请求列表，每个包含请求 ID 和事件回调（on_stream/on_response/on_complete/on_error/on_progress）。
- `pub http_servers: Vec<ScriptHttpServer>` —— 正在监听的 HTTP 服务器列表。每个服务器维护一个 `ToUIReceiver<HttpServerRequest>` 通道以接收来自后端线程的请求。
