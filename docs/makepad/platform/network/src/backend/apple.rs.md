# apple.rs — Apple 平台网络后端

**File path**: `platform/network/src/backend/apple.rs` (167 行)
**Core purpose**: macOS/iOS/tvOS 的 `NetworkBackend` 实现，使用 NSURLSession API 处理 HTTP 请求和 WebSocket 连接。

## 内部枚举

### AppleSocket
- `Plain(PlainWebSocket)` — 基于标准 TCP 的 WebSocket
- `Platform(AppleWebSocket)` — 基于 NSURLSessionWebSocketTask 的原生实现

## AppleBackend

### 字段
- `http_requests: Arc<Mutex<AppleHttpRequests>>` — 管理活跃的 HTTP 请求
- `sockets: Mutex<HashMap<LiveId, AppleSocket>>` — WebSocket 连接表

### NetworkBackend 实现

#### http_start
- 创建 mpsc channel 用于后端响应
- 调用 `AppleHttpRequests::make_http_request()` 发起 NSURLSession 请求
- 在后台线程监听响应并转发到 `EventSink`

#### http_cancel
- 调用 `AppleHttpRequests::cancel_http_request()` 取消特定请求

#### ws_open
- 根据 `WebSocketTransport` 选择 `Plain` 或 `Platform`
- 插入 socket 表，后台线程转发 WS 事件到 EventSink

#### ws_send / ws_close
- 从 socket 表查找并分发操作（支持两种 socket 类型）

## 辅助函数

### map_ws_event(socket_id, message) -> NetworkResponse
将 `WebSocketMessage` 转换为 `NetworkResponse` 枚举值

### create_backend() -> Arc<dyn NetworkBackend>
创建 `AppleBackend` 实例
