# socket_stream.rs — TCP/SSL 套接字流

**文件路径**: `platform/src/os/linux/socket_stream.rs` (387 行)
**核心功能**: 封装 TCP 和 TLS（OpenSSL）连接的套接字流，提供统一的 `Read`/`Write` 接口。

## 主要类型

### `SocketStream`
TCP/TLS 流枚举：
- `Plain(TcpStream)` — 未加密 TCP 连接
- `Tls(OpenSslStream)` — OpenSSL 加密连接

### `OpenSslStream`
OpenSSL 加密流实现：
- `tcp_stream: TcpStream` — 底层 TCP 连接
- `ssl_ctx: *mut SSL_CTX` — SSL 上下文
- `ssl: *mut SSL` — SSL 连接对象

## OpenSSL FFI 绑定

通过 `#[link(name = "ssl")]` 和 `#[link(name = "crypto")]` 静态链接 OpenSSL，包含：
- `OPENSSL_init_ssl` / `TLS_client_method` / `SSL_CTX_new` / `SSL_new`
- `SSL_set_fd` / `SSL_connect` / `SSL_read` / `SSL_write` / `SSL_shutdown`
- `SSL_get_error` / `SSL_ctrl` / `SSL_CTX_set_verify` / `SSL_CTX_set_default_verify_paths`
- `ERR_get_error` / `ERR_error_string_n`

## 关键方法

### `OpenSslStream::connect(tcp_stream, host, verify_peer)`
建立 TLS 连接：
1. 初始化 OpenSSL（单次惰性初始化）
2. 创建 SSL 上下文和 SSL 对象
3. 设置 SNI（Server Name Indication）
4. 将 SSL 绑定到 TCP 文件描述符
5. 握手循环：处理 `WANT_READ`/`WANT_WRITE`（忙等待 1ms）和 `SYSCALL`（检查 WouldBlock）
6. 验证模式可选（`verify_peer` 决定是否验证服务器证书）

### `SocketStream::connect(host, port, use_tls, ignore_ssl_cert)`
连接到远程主机：
1. 通过 `TcpStream::connect` 建立 TCP 连接
2. 启用 `TCP_NODELAY`
3. 如果需要 TLS，创建 `OpenSslStream`；否则使用 `Plain`

### Read/Write 实现
- `SSL_read` / `SSL_write` — 封装 OpenSSL I/O
- 处理 `WANT_READ`/`WANT_WRITE` 返回 `WouldBlock`
- 处理 `SYSCALL` 错误：转换 WouldBlock/TimedOut/Interrupted

## 生命周期

`Drop` 实现中依次调用 `SSL_shutdown`、`SSL_free`、`SSL_CTX_free`、TCP `shutdown`。
