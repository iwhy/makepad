# Android 网络实现

## 概述

`android_network.rs` 实现了 Android 平台的网络功能（HTTP 请求和 WebSocket）。网络操作通过 JNI 桥接到 Java 的 `HttpURLConnection` 和 `WebSocket` 实现，同时支持纯 TCP WebSocket 回退。

## 核心类型

### 内部类型

| 类型 | 说明 |
|------|------|
| `PendingHttp` | 等待中的 HTTP 请求（含 `sink` 回调） |
| `HttpState` | HTTP 请求状态（内部 ID ↔ 公开 ID 映射） |
| `PendingWs` | 等待中的 WebSocket 连接 |
| `AndroidSocket` | WebSocket 类型枚举（Plain/Platform） |
| `AndroidWebSocket` | 平台 WebSocket 包装（Java 桥接） |
| `WsState` | WebSocket 状态（公开 ID ↔ PendingWs 映射 + 解析器） |
| `AndroidNetworkShimBackend` | 网络后端核心实现 |

### IO 类型

| 类型 | 说明 |
|------|------|
| `AndroidSocketStreamFactoryImpl` | 原生 TCP Socket 工厂 |
| `AndroidSocketStreamImpl` | 原生 TCP Socket 流 |

## 关键功能

### 网络后端初始化

`install_network_backend_shim()` — 使用 `Once` 确保只调用一次：
- 创建 `AndroidNetworkShimBackend`
- 注册为 Android 网络后端和 Socket 流工厂

### HTTP 请求流程

```rust
http_start(request_id, request, sink) → 分配 internal_id → 存储映射 → JNI 调用 → Java 处理
```

- `try_handle_http_response` / `try_handle_http_error` — Java 回调入口
- 查找 `by_internal` 映射，通过 `sink.emit()` 分发 `HttpResponse` 或 `HttpError`

### WebSocket 流程

```rust
ws_open(socket_id, request, sink)
```

两种传输模式：
1. **PlainTcp（纯 TCP）**：使用 Rust 实现的 `PlainWebSocket`
2. **Platform（平台）**：通过 JNI 桥接到 Java `WebSocket`

WebSocket 消息通过 `mpsc::channel` 在后台线程中接收，通过 `sink.emit()` 转发。

### 原生 Socket 流

`AndroidSocketStreamImpl` 实现 `AndroidSocketStream` trait：
- `connect` — 通过 JNI 打开 TCP 连接
- `read` / `write` — 通过 JNI 读写
- `set_read_timeout` / `set_write_timeout` — 设置超时
- `shutdown` / `Drop` — 关闭连接

## 公共入口函数

| 函数 | 说明 |
|------|------|
| `try_handle_http_response` | HTTP 响应回调 |
| `try_handle_http_error` | HTTP 错误回调 |
| `try_handle_websocket_message` | WebSocket 消息回调 |
| `try_handle_websocket_closed` | WebSocket 关闭回调 |
| `try_handle_websocket_error` | WebSocket 错误回调 |

## 实现说明

- `AndroidNetworkShimBackend` 使用 `AtomicU64` 生成单调递增的内部请求/连接 ID。
- WebSocket 消息使用 `WebSocketParser` 解析 RFC 6455 帧。
- WebSocket 关闭时自动清理内部映射，防止内存泄漏。
- 纯 TCP WebSocket（`PlainTcp`）适用于需要完全 Rust 端控制的场景。
