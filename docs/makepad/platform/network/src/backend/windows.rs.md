# windows.rs — Windows 平台网络后端

**File path**: `platform/network/src/backend/windows.rs` (149 行)
**Core purpose**: Windows 的 `NetworkBackend` 实现，使用 WinRT HttpClient 处理 HTTP，使用 WinHTTP API 处理 WebSocket。

## 子模块
- `pub mod http` — WinRT HttpClient 实现
- `pub mod web_socket` — WinHTTP WebSocket 实现

## 内部枚举

### WindowsSocket
- `Plain(PlainWebSocket)` — 基于标准 TCP 的 WebSocket
- `Platform(WindowsWebSocket)` — 基于 WinHTTP API 的 WebSocket

## WindowsBackend

### 字段
- `sockets: Mutex<HashMap<LiveId, WindowsSocket>>`

### NetworkBackend 实现

#### http_start
- 创建 mpsc channel
- 调用 `WindowsHttpSocket::open()` 在新线程中执行
- 后台线程转发响应

#### http_cancel
- 当前为空操作（无取消支持）

#### ws_open
- `PlainTcp` → `PlainWebSocket`
- `Platform` → `WindowsWebSocket::open`（WinHTTP）
- 插入 socket 表，后台线程转发 WS 事件

#### ws_send / ws_close
- 从 socket 表查找并分发

### create_backend() -> Arc<dyn NetworkBackend>
创建 `WindowsBackend` 实例
