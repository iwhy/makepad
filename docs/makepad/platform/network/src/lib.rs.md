# lib.rs — Network Crate 根模块

**File path**: `platform/network/src/lib.rs` (48 行)
**Core purpose**: 聚合导出 network crate 的所有公共类型和模块，提供平台特定的 Android/WASM 后端 shim 符号。

## 模块声明

声明以下子模块：
- `backend` — 平台网络后端抽象
- `digest` — SHA1/MD5/SHA256/Base64 无依赖实现
- `http_server` — 嵌入式 HTTP/WebSocket 服务器
- `plain_web_socket` — 纯 TCP WebSocket 客户端
- `runtime` — 网络运行时封装
- `socket_stream` — 跨平台 SocketStream 包装
- `types` — 核心网络类型定义
- `ui_signal` — UI 事件循环信号集成
- `utils` — HTTP 工具函数
- `web_socket_parser` — WebSocket 帧解析器

## 公共 Re-export

### 核心类型
- `NetworkBackend`, `EventSink`, `UnsupportedBackend` — 后端 trait 与事件通道
- `NetworkRuntime`, `NetworkConfig` — 运行时入口
- `SocketStream` — 跨平台 socket
- `HttpRequest`, `HttpResponse`, `HttpError`, `HttpProgress`, `HttpMethod`, `NetworkResponse`, `NetworkError`, `SplitUrl` — HTTP 类型
- `WebSocketTransport`, `WebSocketMessage`, `WsMessage`, `WsSend` — WebSocket 类型
- `HttpServer`, `HttpServerRequest`, `HttpServerResponse` — HTTP 服务器类型

### UI 信号
- `SignalToUI`, `SignalFromUI`, `ToUIReceiver`, `ToUISender`, `FromUIReceiver`, `FromUISender`

### WebSocket 解析器
- `WebSocketParser`, `WebSocketError`, `WebSocketMessage` (parsed), `WebSocketMessageFormat`, `WebSocketMessageHeader`
- 服务器端别名: `ServerWebSocketMessage`, `ServerWebSocketError`, 等
- `SERVER_WEB_SOCKET_PING_MESSAGE`, `SERVER_WEB_SOCKET_PONG_MESSAGE`

### 工具
- `HttpServerHeaders`

## Platform-Specific 导出

### Android
`#[cfg(target_os = "android")]` 下导出 `*_android_backend_shim` / `*_android_socket_stream_factory_shim` 符号，以及 `AndroidSocketStreamFactory` / `AndroidSocketStream` 别名。

### WASM
`#[cfg(target_arch = "wasm32")]` 下导出 `*_wasm_backend_shim` 符号。
