# `net.rs` — 网络标准库（WebSocket / HTTP / Socket 流）

## 文件位置
`platform/script/std/src/net.rs`

---

## 总体职责
这个文件是 Splash 标准库中最复杂的模块，实现了四种网络原语：HTTP 请求、HTTP 服务器、WebSocket 客户端、TCP/TLS Socket 流。它使用 `makepad_network` crate 提供的网络运行时（`NetworkRuntime`）进行异步 I/O 操作，并通过 `ToUIReceiver`/`FromUISender` 通道在后台线程和 UI 线程之间传递事件。

---

## 关键数据结构

### `ScriptWebSocket` / `ScriptServerWebSocket`
- **`ScriptWebSocket`** 表示客户端 WebSocket 连接，包含 `id`（请求 ID）、`socket_id`（Socket ID）、以及 `WebSocketEvents`（`on_opened`/`on_closed`/`on_binary`/`on_string`/`on_error` 回调函数）。
- **`ScriptServerWebSocket`** 表示 HTTP 服务器接受的 WebSocket 连接，额外包含 `web_socket_id: u64`（服务端分配的 ID）和 `response_sender`（用于向该 WebSocket 发送二进制消息的通道发送端）。

### `ScriptHttp` / `ScriptHttpServer`
- **`ScriptHttp`** 表示一个 HTTP 请求，持有 `id` 和 `HttpEvents` 回调对象。
- **`ScriptHttpServer`** 表示一个 HTTP 服务器，包含唯一 `id`、`ToUIReceiver<HttpServerRequest>`（接收来自网络后端的 HTTP 请求）、`HttpServerEvents` 回调、以及该服务器管理的 `Vec<ScriptServerWebSocket>`。

### `ScriptSocketStream` / `ScriptSocketStreamGc`
- **`ScriptSocketStream`** 表示一个 TCP 或 TLS 加密的 Socket 流。包含：
  - `handle: ScriptHandle` —— 脚本侧的 GC 句柄。
  - `host: String` —— 连接的原始主机名（用于 TLS SNI）。
  - `in_send: FromUISender<SocketStreamIn>` —— 用于向后台线程发送写入/StartTls/Close 命令。
  - `out_recv: ToUIReceiver<SocketStreamOut>` —— 用于从后台线程接收数据/错误/关闭事件。
  - `recv_pause: VecDeque<ScriptThreadId>` —— 暂停等待数据的协程线程 ID 队列。
  - `pending_chunks: VecDeque<Vec<u8>>` —— 已接收但尚未被脚本消费的数据块队列。
  - `is_closed: bool` / `last_error: Option<String>` —— 连接状态。
- **`ScriptSocketStreamGc`** 实现 `ScriptHandleGc` trait，在 GC 回收句柄时自动关闭 socket 流并发送 `Close` 命令。

### `SocketStreamIn` / `SocketStreamOut` / `SocketStreamPoll`
- `SocketStreamIn` 是后台线程命令枚举：`Write(Vec<u8>)`、`StartTls { host, ignore_ssl_cert }`、`Close`。
- `SocketStreamOut` 是后台线程事件枚举：`Data(Vec<u8>)`、`Error(String)`、`Closed`。
- `SocketStreamPoll` 是脚本协程轮询结果枚举：`Data(Vec<u8>)`、`Closed(Option<String>)`、`Pause`、`TooManyPaused`、`InvalidHandle`。

### 事件回调结构体
- **`HttpServerEvents`**：`on_get` / `on_post` / `on_connect_websocket` 三个可选回调。
- **`HttpEvents`**：`on_stream` / `on_response` / `on_complete` / `on_error` / `on_progress` 五个可选回调。
- **`WebSocketEvents`**：`on_opened` / `on_closed` / `on_binary` / `on_string` / `on_error` 五个可选回调。
- **`SocketStreamOptions`**：`host: String` / `port: String` / `use_tls: bool` / `ignore_ssl_cert: bool` / `read_timeout_ms: f64` / `write_timeout_ms: f64`。

---

## 关键函数（Rust 侧工具）

### `fn socket_stream_thread(...)`
- 这是一个独立的无限循环线程函数，运行在后台线程中。它负责：
  1. 轮询 `in_recv` 通道上的命令（`Write`/`StartTls`/`Close`）。
  2. 使用非阻塞方式读取 socket 数据（`socket.read(&mut read_buf)`，16KB 缓冲区）。
  3. 将读取到的数据通过 `out_send` 发送回 UI 线程。
  4. 处理 `WouldBlock`/`TimedOut`/`Interrupted` 错误（视为正常，继续轮询），其他错误则发送 `Error` + `Closed` 后退出线程。
- **StartTls 处理**：将原始 TCP socket 通过 `socket.into_tls(&current_host, ignore_ssl_cert)` 升级为 TLS socket。可以指定覆盖的 host 值。
- **Close 处理**：调用 `socket.shutdown()` 优雅关闭连接，然后发送 `Closed` 事件并结束线程。

### `fn script_value_to_bytes(vm, value) -> Result<Vec<u8>, String>`
- 将 Splash 脚本值转换为字节数组。支持三种输入类型：
  1. **字符串类型**：直接提取 UTF-8 字节序列。
  2. **数组类型**：依据底层存储类型处理——`U8` 直接克隆、`U16`/`U32` 逐元素 `as u8` 截断、`F32` 逐元素截断、`ScriptValue` 逐元素 `as_f64()` 并检查 0..=255 范围后截断。
  3. **其他类型**：返回 `"expected string or byte array"` 错误。

### `fn socket_stream_index(vm, handle) -> Option<usize>`
- 在 `ScriptStd` 的 `socket_streams` 列表中查找与给定 `handle` 匹配的流的索引位置。

### `pub fn socket_stream_send_bytes(vm, handle, data) -> Result<(), String>`
- 向指定 handle 的 socket 流发送原始字节数据。内部通过 `in_send.send(SocketStreamIn::Write(data))` 发送。
- 如果 handle 无效返回 "invalid socket_stream handle"，如果通道关闭返回 "socket stream is closed"。

### `pub fn socket_stream_poll(vm, handle) -> SocketStreamPoll`
- 轮询指定 socket 流的数据状态。检查优先级为：待处理块 → 已关闭 → 暂停队列超过 100 个 → 返回 `Pause`。
- 如果超过 100 个协程暂停在此 socket 上等待，返回 `TooManyPaused` 防止无限增长。

### `pub fn socket_stream_pause_current(vm, handle) -> Result<(), String>`
- 暂停当前协程线程，并将线程 ID 添加到 socket 流的 `recv_pause` 队列前部（LIFO 行为）。
- 当新数据到达时，`handle_script_socket_streams` 会从队列尾部取出线程 ID 继续执行。

---

## 事件泵函数

### `pub fn handle_script_socket_streams(host, std, script_vm)`
- 遍历 `std.data.socket_streams` 中所有流，尝试从每个流的 `out_recv` 通道 recv 消息。
- 数据消息：推到 `pending_chunks` 队列，从 `recv_pause` 弹出一个暂停的线程 ID 加入 `resume_threads` 列表。
- 错误/关闭消息：标记 `is_closed`，排出所有暂停线程。
- 收集到的 `resume_threads` 统一调用 `task::queue_script_thread_resume` 放入待恢复队列。

### `pub fn handle_script_http_servers(host, std, script_vm)`
- 遍历 `http_servers`，从每个服务器的 `receiver` 通道接收 `HttpServerRequest` 消息。
- 处理消息类型（从 Get/Post/WebSocket 到 BinaryMessage 等），通过 `vm::with_vm_and_async` 在 VM 上下文中调用对应的 Splash 回调函数。
- **Get/Post 处理**：将 HTTP headers 和 body 转换为脚本值，调用回调后等待 `HttpServerResponse` 对象返回，然后通过 `response_sender.send(response)` 发送响应。如果回调未返回 `HttpServerResponse`，发送默认的 "200 OK" 响应。
- **WebSocket 连接**：调用 `on_connect_websocket` 回调，检查返回值是否包含 `WebSocketEvents` 协议，如果是则创建 `ScriptServerWebSocket` 加入到服务器列表。
- **WebSocket 消息**：根据消息类型（BinaryMessage/TextMessage）调用对应的 `on_binary`/`on_string` 回调。
- **DisconnectWebSocket**：调用 `on_closed` 回调，然后从服务器的 WebSocket 列表中移除该连接。

### `pub fn handle_script_web_socket_event(host, std, script_vm, event: NetworkResponse)`
- 处理来自网络运行时的 WebSocket 事件（`WsOpened`/`WsMessage`/`WsClosed`/`WsError`）。
- 在 `web_sockets` 列表中查找匹配 `socket_id` 的项目，然后调用对应的回调函数。
- 对于 `WsClosed` 和 `WsError`，处理回调后从列表中移除该 WebSocket。

### `pub fn handle_script_network_events(host, std, script_vm, responses: &[NetworkResponse])`
- 批量处理网络运行时事件。先过滤出所有 WebSocket 事件交给 `handle_script_web_socket_event` 处理。
- 然后处理 HTTP 相关事件：`HttpStreamChunk`（调用 on_stream）、`HttpStreamComplete`（调用 on_complete 后移除请求）、`HttpResponse`（调用 on_response 后移除请求）、`HttpError`（调用 on_error 后移除请求）。
- 所有 HTTP 事件处理完毕后都会将对应的 `ScriptHttp` 从列表中移除，确保不会重复处理。

### `pub fn drain_network_runtime(std: &mut ScriptStd) -> Vec<NetworkResponse>`
- 从 `std.net`（`NetworkRuntime`）中通过 `try_recv()` 排干所有待处理的网络事件。
- 如果 `std.net` 为 `None`，直接返回空 `Vec`。
- 该函数被 `lib.rs` 的 `pump_network_runtime` 调用，用于在自定义泵循环中收集事件。

---

## 脚本模块注册

### `pub fn script_mod(vm: &mut ScriptVm)`
- 创建 `mod.net` 模块，注册类型和方法的入口。

**注册的类型 API：**
- 使用 `set_script_value_to_api!` 注册：`HttpRequest`、`HttpMethod`、`HttpEvents`、`HttpServerEvents`、`HttpServerOptions`、`HttpServerResponse`、`HttpServerHeaders`、`WebSocketEvents`、`SocketStreamOptions`。

**`net.http_server(options, events)`：**
- 请求类型检查（`HttpServerOptions` + `HttpServerEvents`），使用 `script_has_proto!` 验证。
- 通过标准的 `channel()` + `ToUIReceiver` 创建 UI 线程通道。
- 启动后台线程将 `HttpServer` 的请求转发到 UI 线程。
- 调用 `runtime.start_http_server(server)` 在 `NetworkRuntime` 中注册服务器。
- 为服务器生成唯一的 `LiveId`，将 `ScriptHttpServer` 加入 `std.data.http_servers`。

**`net.http_request(request, events)`：**
- 类型检查后调用 `runtime.http_start(id, request)` 发起 HTTP 请求。
- 若失败抛出 "http request failed: {err}"。

**`net.web_socket(request, events)`：**
- 支持字符串 URL（自动构造默认 `HttpRequest`）和对象形式。
- 调用 `runtime.ws_open(id, request)` 打开 WebSocket 连接。
- 若连接失败返回 `NIL`。

**`net.socket_stream(options)`：**
- 验证 host 和 port 非空后，调用 `SocketStream::connect(...)` 建立连接。
- 设置读写超时时间（options 中的 read_timeout_ms/write_timeout_ms，为 0 表示无超时）。
- 启动后台线程运行 `socket_stream_thread`。
- 创建 `ScriptSocketStreamGc` GC 回调，通过 `vm.bx.heap.new_handle(socket_stream_type, gc_box)` 创建 `ScriptHandle`。
- 将 `ScriptSocketStream` 实例加入 `std.data.socket_streams`。
- 返回 `handle.into()` 给脚本侧。

**Socket 流句柄属性和方法：**
- 属性 getter：`closed`（bool）、`pending`（f64，待消费块数）、`error`（Option<string>）、`host`（string）。
- 方法：
  - `write(data)` —— 写入原始字节或字符串，返回写入的字节数。
  - `write_string(data)` —— 写入字符串，返回写入的字节数。
  - `start_tls(host?, ignore_ssl_cert?)` —— 升级为 TLS 连接。
  - `close()` —— 关闭连接。
  - `next()` —— 消费一个数据块（返回 `Uint8Array`），或暂停等待。
  - `next_string()` —— 消费一个数据块（返回 `String`），或暂停等待。
