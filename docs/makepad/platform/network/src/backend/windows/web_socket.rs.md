# windows/web_socket.rs — Windows WinHTTP WebSocket 客户端

**File path**: `platform/network/src/backend/windows/web_socket.rs` (572 行)
**Core purpose**: 使用 `winhttp.dll` 的 WinHTTP WebSocket API 实现的 Windows 原生 WebSocket 客户端。

## FFI 声明

链接 `winhttp.dll` 和 `kernel32.dll`：
- `WinHttpOpen`, `WinHttpConnect`, `WinHttpOpenRequest`
- `WinHttpSetOption`, `WinHttpSendRequest`, `WinHttpReceiveResponse`
- `WinHttpWebSocketCompleteUpgrade`, `WinHttpWebSocketSend/Receive/Close`
- `WinHttpCloseHandle`, `GetLastError`

## 常量

### WebSocket 缓冲区类型
- `WINHTTP_WEB_SOCKET_BINARY_MESSAGE_BUFFER_TYPE = 0`
- `WINHTTP_WEB_SOCKET_BINARY_FRAGMENT_BUFFER_TYPE = 1`
- `WINHTTP_WEB_SOCKET_UTF8_MESSAGE_BUFFER_TYPE = 2`
- `WINHTTP_WEB_SOCKET_UTF8_FRAGMENT_BUFFER_TYPE = 3`
- `WINHTTP_WEB_SOCKET_CLOSE_BUFFER_TYPE = 4`

### 安全标志
- `SECURITY_FLAG_IGNORE_UNKNOWN_CA`, `IGNORE_CERT_WRONG_USAGE`
- `IGNORE_CERT_CN_INVALID`, `IGNORE_CERT_DATE_INVALID`

## 内部函数

### wide_null(s) -> Vec<u16>
将字符串编码为 UTF-16 null-terminated

### open_winhttp_websocket(request) -> Result<Arc<WinHttpWebSocket>, String>
- `WinHttpOpen` → `WinHttpConnect` → `WinHttpOpenRequest`
- TLS + `ignore_ssl_cert` → `WinHttpSetOption(SECURITY_FLAGS)`
- `WinHttpSetOption(UPGRADE_TO_WEB_SOCKET)`
- 发送请求 → 接收响应 → `WinHttpWebSocketCompleteUpgrade`

## WinHttpWebSocket

### 字段
- `session, connect, websocket: *mut c_void` — WinHTTP 句柄
- `closed: AtomicBool`

### send_message(message) -> Result<(), u32>
- 发送 `Binary` / `String` / `Close` 帧
- 通过 `WinHttpWebSocketSend` 发送

### receive(buffer) -> Result<(usize, u32), u32>
- 通过 `WinHttpWebSocketReceive` 读取帧
- 返回 (字节数, 缓冲区类型)

### shutdown() / Drop
- 原子 `closed` 保护，清理所有 WinHTTP 句柄

## WindowsWebSocket

### open(socket_id, request, rx_sender) -> Self
- 调用 `open_winhttp_websocket` 建立连接
- 写线程：从 mpsc 读取出站消息并通过 `WinHttpWebSocket::send_message` 发送
- 读线程：循环 `WinHttpWebSocket::receive`
  - 处理分片（`*_FRAGMENT_*` 类型累积，`*_MESSAGE_*` 类型发送完整消息）
  - 支持 UTF-8 文本分片重组
  - `CLOSE_BUFFER_TYPE` → 发送 `WebSocketMessage::Closed`
  - 错误 → 发送 `Error` + `Closed`

### send_message / close
- 委托给 mpsc sender 和 WinHttpWebSocket 的 shutdown
