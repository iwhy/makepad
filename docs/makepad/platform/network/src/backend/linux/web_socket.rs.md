# linux/web_socket.rs — Linux OpenSSL WebSocket 客户端

**File path**: `platform/network/src/backend/linux/web_socket.rs` (305 行)
**Core purpose**: 基于 OpenSSL `SocketStream` 的 Linux WebSocket 客户端实现，支持 TLS/非 TLS 连接。

## LinuxWebSocket

### 字段
- `sender: Option<Sender<WebSocketMessage>>` — I/O 线程通信通道
- 不持有 stream 引用（I/O 线程拥有所有权）

### open(socket_id, request, rx_sender) -> Self
- 解析 URL 确定是否 TLS（ws/http → 否，wss/https → 是）
- 使用 `SocketStream::connect` 建立连接（支持 TLS）
- 设置 50ms 读超时、30s 写超时
- 构造 HTTP 升级请求并发送
- 读取 WebSocket 握手响应（5 秒超时）
- 启动 I/O 线程：
  - 从 mpsc receiver 取出站消息并编码为 WS 帧
  - TCP 读取 → `WebSocketParser` 解析
  - Ping → 自动回复 Pong
  - 错误/关闭 → 发送事件并终止

### send_message(message) -> Result<(), ()>
- 向 I/O 线程发送消息

### close()
- 断开 sender，I/O 线程将检测并终止

## 内部函数

### handle_outgoing_message(stream, msg) -> bool
- 编码并发送 Binary/Text/Closed 帧

### parse_incoming(parser, stream, rx_sender, done, bytes)
- 解析 WS 帧并通过 mpsc 转发

### read_websocket_handshake_response(stream) -> Result<Vec<u8>, String>
- 读取 HTTP 响应直到 `\r\n\r\n`，验证状态为 101

### write_all_no_error(stream, bytes) -> bool
- 完整写入，处理阻塞错误

### find_header_end(data) -> Option<usize>
- 查找 `\r\n\r\n`
