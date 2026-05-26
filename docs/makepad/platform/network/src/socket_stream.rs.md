# socket_stream.rs — 跨平台 SocketStream 包装

**File path**: `platform/network/src/socket_stream.rs` (300 行)
**Core purpose**: 提供平台无关的 `SocketStream` 类型，将各平台 socket 实现统一为 `Read + Write` 接口，支持 TLS 升级。

## 架构

`SocketStream` 是一个条件编译的包装结构体，对每个目标平台使用不同的内部实现：

| 目标 | 内部类型 |
|------|---------|
| Linux (`target_os = "linux"`) | `crate::backend::linux::socket_stream::SocketStream` |
| macOS/iOS/tvOS | `crate::backend::apple::socket_stream::SocketStream` |
| Windows | `crate::backend::windows::socket_stream::SocketStream` |
| Android | `Box<dyn PlatformSocketStream>` (trait object) |
| WASM | 空结构体，所有操作返回 `Unsupported` |

## 公有 API

所有平台一致的接口：

- `connect(host, port, use_tls, ignore_ssl_cert) -> io::Result<Self>` — 建立 TCP + 可选 TLS 连接
- `into_tls(host, ignore_ssl_cert) -> io::Result<Self>` — 在已有连接上升级 TLS（Android/WASM 不支持）
- `set_read_timeout(timeout)`, `set_write_timeout(timeout)` — 设置超时
- `shutdown()` — 关闭连接

实现 `Read` 和 `Write` trait，可透明地在需要 `io::Read` / `io::Write` 的泛型代码中使用。
