# plain_web_socket.rs — 纯 TCP WebSocket 客户端

**File path**: `platform/network/src/plain_web_socket.rs` (343 行)
**Core purpose**: 基于标准 `TcpStream` 的 WebSocket 客户端实现，手动完成 HTTP 升级握手和 WebSocket 帧 I/O。

## 结构体

### PlainWebSocket
- `sender: Option<Sender<WebSocketMessage>>` — 向 I/O 线程发送出站消息
- `stream: Option<TcpStream>` — TCP 连接（仅用于 Drop 时关闭）
- Drop 时关闭流并断开 sender

## 关键方法

### open(socket_id, request, rx_sender) -> Self
- 解析 URL 获取协议、主机、端口
- 仅支持 `ws://` 和 `http://`（不支持 TLS — 会直接返回错误）
- 连接 TCP，设置 nodelay 和 50ms 读取超时
- 构造并发送 HTTP WebSocket 升级请求
- 读取升级响应（5 秒超时），验证 `HTTP/1.1 101`
- 启动 I/O 线程：
  - 从 mpsc receiver 读取出站消息并写入 WS 帧
  - 从 TCP 流读取数据并解析 WS 帧
  - Ping → 自动回复 Pong
  - 错误/关闭 → 发送相应事件并退出

### send_message(message) -> Result<(), ()>
- 向 I/O 线程发送消息

### close()
- 断开 sender 并关闭 TCP 流

## 内部函数

### handle_outgoing_message(stream, msg) -> bool
- 将消息编码为 WebSocket 帧：Binary / Text 带 header，Closed 返回 true 终止

### parse_incoming(parser, stream, rx_sender, done, bytes)
- 解析 WebSocket 帧并通过 mpsc sender 转发到调用方

### read_websocket_handshake_response(stream) -> Result<Vec<u8>, String>
- 读取 HTTP 升级响应并返回 leftover data

### write_all_no_error(stream, bytes) -> bool
- 完整写入，处理 WouldBlock/超时/中断

### find_header_end(data) -> Option<usize>
- 查找 `\r\n\r\n`
