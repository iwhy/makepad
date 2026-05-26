# apple/socket_stream.rs — Apple SecureTransport SocketStream

**File path**: `platform/network/src/backend/apple/socket_stream.rs` (354 行)
**Core purpose**: 基于 Apple 的 `SecureTransport` 框架实现 TLS socket 流，提供 `Read + Write` 兼容的 `SocketStream` 类型。

## 常量
- `SSL_OK = 0`, `ERR_SSL_WOULD_BLOCK = errSSLWouldBlock`
- `ERR_SSL_CLOSED_GRACEFUL`, `ERR_SSL_CLOSED_ABORT`

## SecureTransport TLS 回调

### ssl_read_callback / ssl_write_callback
- 传递给 `SSLSetIOFuncs` 的 C 回调函数
- 桥接 `SSLContext` 和 `TcpStream` 之间的 I/O
- 处理 `WouldBlock` / `TimedOut` / `Interrupted` 返回 `errSSLWouldBlock`

## SecureTransportStream

### 字段
- `tcp_stream: Box<TcpStream>`, `ssl_context: SSLContextRef`, `is_closed: bool`
- 实现 `Send`（手动标记，仅从单一线程访问）

### connect(tcp_stream, host, verify_peer)
- 调用 `SSLCreateContext(kSSLClientSide, kSSLStreamType)` 创建 SSL 上下文
- 设置 I/O 回调和连接引用
- 设置 SNI 主机名 (`SSLSetPeerDomainName`)
- `verify_peer = false` → 设置 `kSSLSessionOptionBreakOnServerAuth` 跳过验证
- 循环调用 `SSLHandshake` 直到完成
- 错误时清理并返回

### 其他方法
- `set_read_timeout`, `set_write_timeout` — 委托给 `tcp_stream`
- `shutdown` — `SSLClose` + `CFRelease` + 关闭 TCP

### Read / Write 实现
- `SSLRead` / `SSLWrite` 包装，处理返回值到 `io::Result` 的映射

## SocketStream (enum)

- `Plain(TcpStream)` — 非 TLS
- `Tls(SecureTransportStream)` — TLS 加密

### 方法
- `connect(host, port, use_tls, ignore_ssl_cert)` — 自动选择
- `into_tls(host, ignore_ssl_cert)` — 从 Plain 升级到 Tls
- `set_read_timeout`, `set_write_timeout`, `shutdown` — 分发到内部类型
- 实现 `Read` 和 `Write` trait
