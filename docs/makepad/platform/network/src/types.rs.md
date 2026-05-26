# types.rs — 核心网络类型定义

**File path**: `platform/network/src/types.rs` (392 行)
**Core purpose**: 定义 HTTP/WebSocket 请求、响应、错误及进度等核心数据类型，支持 `script` feature 下的脚本集成。

## 枚举

### WebSocketTransport
- `Auto` (default) — 自动选择传输方式
- `PlainTcp` — 强制纯 TCP (native WebSocket 不走平台 API)
- `Platform` — 强制使用平台原生 WebSocket API

### HttpMethod
- `GET`, `HEAD`, `POST`, `PUT`, `DELETE`, `CONNECT`, `OPTIONS`, `TRACE`, `PATCH`
- 方法 `as_str()` / `to_string()` 返回 HTTP 方法字符串

### WebSocketMessage
- `Error(String)`, `Binary(Vec<u8>)`, `String(String)`, `Opened`, `Closed`

### WsSend / WsMessage
- `WsSend`: `Binary(Vec<u8>)`, `Text(String)` — 发送方向
- `WsMessage`: `Binary(Vec<u8>)`, `Text(String)` — 接收方向

### NetworkResponse
- `HttpResponse { request_id, response }` — 完整 HTTP 响应
- `HttpStreamChunk { request_id, response }` — 流式分块
- `HttpStreamComplete { request_id, response }` — 流式结束
- `HttpError { request_id, error }` — 错误
- `HttpProgress { request_id, progress }` — 进度
- `WsOpened { socket_id }`, `WsMessage { socket_id, message }`, `WsClosed { socket_id }`, `WsError { socket_id, message }` — WebSocket 事件

### NetworkError
- `Unsupported(&'static str)` — 平台不支持
- `Backend(String)` — 后端错误
- `ChannelClosed` — 通道关闭
- 实现 `Display` 和 `std::error::Error`

## 结构体

### HttpRequest
- `metadata_id: LiveId`, `url: String`, `method: HttpMethod`
- `headers: BTreeMap<String, Vec<String>>`, `ignore_ssl_cert: bool`
- `is_streaming: bool`, `body: Option<Vec<u8>>`, `websocket_transport: WebSocketTransport`
- 构造器 `new(url, method)` + 多个 setter: `set_ignore_ssl_cert`, `set_is_streaming`, `set_metadata_id`, `set_header`, `set_body`, `set_body_string`, `set_json_body`, `set_string_body`, `set_websocket_transport`
- `get_headers_string()` — 序列化为 `Key: Value\r\n` 格式
- `split_url()` — 解析 URL 返回 `SplitUrl`

### SplitUrl<'a>
- `proto`, `host`, `port`, `file`, `hash` 字段

### HttpResponse
- `metadata_id`, `status_code`, `headers`, `body`
- `new()` 构造器, `from_header_string()` 从原始 header 字符串解析
- 访问器: `body()`, `body_string()`, `get_body()`, `get_string_body()`, `get_json_body()`
- `set_header()` 添加响应头

### HttpError
- `message: String`, `metadata_id: LiveId`

### HttpProgress
- `loaded: u64`, `total: u64`

## 内部函数
- `parse_headers(header_string)` — 将 `Key: Value` 文本解析为 `BTreeMap`
