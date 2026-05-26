# web_network.rs — Web 网络后端 Shim（HTTP + WebSocket）

**文件路径**: `/home/ubuntu/_github/makepad/platform/src/os/web/web_network.rs` (531 行)
**核心作用**: 通过 WASM FFI 调用 JavaScript 网络 API 实现 HTTP 请求/响应、WebSocket 连接/消息收发，作为 `NetworkBackend` trait 的 Web Shim 实现。

## FFI 外部函数声明

### HTTP

- `js_network_http_request` — 发起 HTTP 请求（含 URL、方法、头、体）
- `js_network_http_cancel` — 取消 HTTP 请求

### WebSocket

- `js_network_ws_open` — 打开 WebSocket 连接
- `js_network_ws_send_binary` — 发送二进制数据
- `js_network_ws_send_text` — 发送文本数据
- `js_network_ws_close` — 关闭 WebSocket 连接

## 内部数据结构

### `PendingHttp`

| 字段 | 说明 |
|------|------|
| `public_request_id` | 对外暴露的请求 ID |
| `sink: EventSink` | 事件回调接收器 |

### `HttpState`

| 字段 | 说明 |
|------|------|
| `by_internal: HashMap<LiveId, PendingHttp>` | 内部 ID → 待处理请求 |
| `by_public: HashMap<LiveId, LiveId>` | 公开 ID → 内部 ID 映射 |

### `PendingWs` / `WsState`

结构与 HTTP 对应，管理 WebSocket 的 ID 映射和事件接收器。

### `WasmNetworkShimBackend`

| 字段 | 说明 |
|------|------|
| `next_internal_id: AtomicU64` | 内部 ID 生成器（高位设 bit 62 避免冲突） |
| `http: Mutex<HttpState>` | HTTP 待处理请求状态 |
| `ws: Mutex<WsState>` | WebSocket 待处理请求状态 |

## `NetworkBackend` Trait 实现

### `http_start`

1. 生成内部 ID，注册到 `by_internal` 和 `by_public`
2. 通过 FFI 调用 `js_network_http_request`

### `http_cancel`

1. 通过内部 ID 查找并移除待处理请求
2. 调用 `js_network_http_cancel`

### `ws_open`

1. 生成内部 ID，注册 WebSocket 状态
2. 调用 `js_network_ws_open`（自动替换 http→ws 和 https→wss 协议）

### `ws_send`

1. 查找内部 ID
2. 根据消息类型调用 `js_network_ws_send_binary` 或 `js_network_ws_send_text`

### `ws_close`

1. 查找并移除 WebSocket 状态
2. 调用 `js_network_ws_close`

## WASM 导出函数（JS→Rust 回调）

| 导出名 | 触发时机 | 处理 |
|--------|---------|------|
| `wasm_network_http_response` | HTTP 响应到达 | `emit_http_response` → 通过 EventSink 发送 `NetworkResponse::HttpResponse` |
| `wasm_network_http_error` | HTTP 请求错误 | `emit_http_error` → `NetworkResponse::HttpError` |
| `wasm_network_http_progress` | HTTP 下载进度 | `emit_http_progress` → `NetworkResponse::HttpProgress` |
| `wasm_network_ws_opened` | WebSocket 连接建立 | `emit_ws_opened` → `NetworkResponse::WsOpened` |
| `wasm_network_ws_closed` | WebSocket 连接关闭 | `emit_ws_closed` → `NetworkResponse::WsClosed` |
| `wasm_network_ws_error` | WebSocket 错误 | `emit_ws_error` → `NetworkResponse::WsError` |
| `wasm_network_ws_text` | WebSocket 文本消息 | `emit_ws_text` → `NetworkResponse::WsMessage(Text)` |
| `wasm_network_ws_binary` | WebSocket 二进制消息 | `emit_ws_binary` → `NetworkResponse::WsMessage(Binary)` |

各 `emit_*` 方法均通过内部 ID 查找对应的公开 ID 和 `EventSink`，通过 `EventSink::emit` 发送响应事件。HTTP 响应使用 `HttpResponse::from_header_string` 解析状态码和头信息。

## 初始化

```rust
pub(crate) fn install_network_backend_shim()
```

使用 `Once` 确保单次初始化，创建 `WasmNetworkShimBackend` 并通过 `register_wasm_backend_shim` 注册到 `makepad_network` 系统。

## 内部 ID 生成

```rust
fn next_internal_id(&self) -> LiveId {
    let raw = self.next_internal_id.fetch_add(1, Ordering::Relaxed) | (1u64 << 62);
    LiveId(raw)
}
```
高 2 位设 `01` 前缀以避免与用户生成的 ID 冲突。

## 数据流

```
Rust: http_start() → 注册 PendingHttp → js_network_http_request()
JS: 网络请求完成 → wasm_network_http_response() → emit_http_response() → EventSink::emit()
```
