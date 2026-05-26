# linux.rs — Linux 平台网络后端

**File path**: `platform/network/src/backend/linux.rs` (156 行)
**Core purpose**: Linux 的 `NetworkBackend` 实现，使用 OpenSSL socket 流处理 HTTP 请求和 WebSocket 连接。

## 子模块
- `pub mod http` — HTTP 客户端
- `pub(crate) mod socket_stream` — OpenSSL socket 流（内部使用）
- `pub mod web_socket` — WebSocket 客户端

## 内部枚举

### LinuxSocket
- `Plain(PlainWebSocket)` — 基于标准 TCP 的 WebSocket
- `Platform(LinuxWebSocket)` — 基于 OpenSSL 连接的 WebSocket

## LinuxBackend

### 字段
- `sockets: Mutex<HashMap<LiveId, LinuxSocket>>`

### NetworkBackend 实现

#### http_start
- 创建 mpsc channel
- 调用 `LinuxHttpSocket::open()` 在新线程中执行 HTTP 请求
- 后台线程转发响应到 EventSink

#### http_cancel
- 调用 `LinuxHttpSocket::cancel()` 设置取消标志

#### ws_open
- 根据 `WebSocketTransport` 选择：
  - `PlainTcp` → 使用 `PlainWebSocket`
  - `Platform` → 使用 `LinuxWebSocket`
  - `Auto` → 基于 URL scheme（ws/http → Plain, wss/https → Platform）
- 插入 socket 表，后台线程转发事件

#### ws_send / ws_close
- 从 socket 表查找并分发操作

### create_backend() -> Arc<dyn NetworkBackend>
创建 `LinuxBackend` 实例
