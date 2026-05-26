# web_socket.rs — WebSocket 实现（JS FFI）

**文件路径**: `/home/ubuntu/_github/makepad/platform/src/os/web/web_socket.rs` (123 行)
**核心作用**: Web 平台 WebSocket 连接管理，通过 WASM FFI 调用 JavaScript WebSocket API 实现双向通信。

## 数据结构

### `OsWebSocket`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `u64` | WebSocket 连接的唯一标识符 |

### 全局 WebSocket 列表

```rust
static WEBSOCKET_LIST: Mutex<RefCell<Vec<(u32, Sender<WebSocketMessage>)>>>
```

全局静态变量，存储所有活跃 WebSocket 连接的 ID 和对应的 MPSC 发送端。

## FFI 外部函数

- `js_open_web_socket(id, url_ptr, url_len)` — 在 JS 侧创建 WebSocket 连接
- `js_web_socket_send_string(id, str_ptr, url_len)` — 发送文本消息
- `js_web_socket_send_binary(id, bin_ptr, bin_len)` — 发送二进制消息

## WASM 导出函数（JS→Rust 回调）

| 导出名 | 触发时机 | 处理 |
|--------|---------|------|
| `wasm_web_socket_closed` | 连接关闭 | 从列表移除，发送 `WebSocketMessage::Closed` |
| `wasm_web_socket_opened` | 连接建立 | 发送 `WebSocketMessage::Opened` |
| `wasm_web_socket_error` | 连接错误 | 发送 `WebSocketMessage::Error`（含错误消息） |
| `wasm_web_socket_string` | 收到文本消息 | 发送 `WebSocketMessage::String`（含文本内容） |
| `wasm_web_socket_binary` | 收到二进制消息 | 发送 `WebSocketMessage::Binary`（含字节数据） |

所有回调均通过 `WEBSOCKET_LIST` 查找对应 ID 的发送通道，并调用 `SignalToUI::set_ui_signal()` 通知 UI 线程。

## `OsWebSocket` 方法

### `open` — 创建新 WebSocket 连接

```rust
pub fn open(id: u64, request: HttpRequest, rx_sender: Sender<WebSocketMessage>) -> OsWebSocket
```

1. 将 (id, sender) 注册到全局列表
2. 自动替换 URL 协议：`https://` → `wss://`，`http://` → `ws://`
3. 通过 FFI 调用 `js_open_web_socket` 在 JS 侧建立连接

### `send_message` — 发送消息

根据消息类型调用 `js_web_socket_send_string` 或 `js_web_socket_send_binary`。

### `close` — 关闭连接

当前为空操作（WebSocket 关闭由 JS 侧处理）。

## 与 `web_network.rs` 的关系

此文件提供较原始的 WebSocket 实现，而 `web_network.rs` 中的 `WasmNetworkShimBackend` 提供了更完整的 `NetworkBackend` trait 封装（含 HTTP 和 WebSocket）。两者均暴露为 `#[export_name]` 的回调入口供 JS 调用。
