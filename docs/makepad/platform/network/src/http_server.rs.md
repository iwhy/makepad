# http_server.rs — 嵌入式 HTTP/WebSocket 服务器

**File path**: `platform/network/src/http_server.rs` (306 行)
**Core purpose**: 使用标准 `TcpListener` 实现的轻量级 HTTP 服务器，支持 GET、POST 和 WebSocket 升级。

## 结构体

### HttpServer
- `listen_address: SocketAddr` — 监听地址
- `request: Sender<HttpServerRequest>` — 请求事件发送到主应用
- `post_max_size: u64` — POST body 大小限制

### HttpServerResponse
- `header: String`, `body: Vec<u8>` — HTTP 响应头和 body

### HttpServerRequest
- `ConnectWebSocket { web_socket_id, headers, response_sender }` — WS 连接请求
- `DisconnectWebSocket { web_socket_id }` — 断开通知
- `BinaryMessage { web_socket_id, response_sender, data }` — 收到的二进制消息
- `TextMessage { web_socket_id, response_sender, string }` — 收到的文本消息
- `Get { headers, response_sender }` — GET 请求
- `Post { headers, body, response }` — POST 请求

## 关键函数

### start_http_server
- 绑定 TCP listener，每连接启动一个线程处理
- 解析 HTTP 请求头，判断操作类型：
  - 含 `Sec-WebSocket-Key` → 升级为 WebSocket
  - `POST` → 读取 body 并发送 Post 事件
  - `GET` → 发送 Get 事件
- 返回线程 `JoinHandle`（None 表示绑定失败）

### WebSocket 处理 (handle_web_socket)
- 完成 HTTP 升级握手（发送 101 Switching Protocols）
- 禁用 Nagle 算法（`set_nodelay(true)`）
- 写线程：从 mpsc channel 读取数据并发送 WS 帧（2 秒超时发 Ping）
- 读线程：解析 WS 帧，处理 Ping/Pong/Text/Binary/Close

### POST 处理 (handle_post)
- 按 `Content-Length` 读取 body
- 验证大小不超过 `post_max_size`

### GET 处理 (handle_get)
- 简单转发请求到主应用 channel
