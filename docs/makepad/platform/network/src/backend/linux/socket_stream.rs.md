# linux/socket_stream.rs — Linux OpenSSL SocketStream

**File path**: `platform/network/src/backend/linux/socket_stream.rs` (398 行)
**Core purpose**: 通过 `libssl` / `libcrypto` C 绑定实现 Linux 平台的 TLS socket 流，提供 `Read + Write` 兼容的 `SocketStream`。

## FFI 声明

链接 `libssl` 和 `libcrypto`：
- `OPENSSL_init_ssl`, `TLS_client_method`, `SSL_CTX_new/free`
- `SSL_new/free`, `SSL_set_fd`, `SSL_connect`, `SSL_get_error`
- `SSL_read/write`, `SSL_shutdown`, `SSL_ctrl` (SNI)
- `SSL_CTX_set_verify`, `SSL_CTX_set_default_verify_paths`
- `ERR_get_error`, `ERR_error_string_n`

## OpenSslStream

### 字段
- `tcp_stream: TcpStream`, `ssl_ctx: *mut SSL_CTX`, `ssl: *mut SSL`

### init_openssl() -> `io::Result<()>`
- 使用 `OnceLock` 确保一次性初始化 `OPENSSL_init_ssl`

### connect(tcp_stream, host, verify_peer)
- 初始化 OpenSSL
- 创建 SSL_CTX 和 SSL 对象
- `verify_peer = true` → `SSL_VERIFY_PEER` + `SSL_CTX_set_default_verify_paths`
- `verify_peer = false` → `SSL_VERIFY_NONE`
- 设置 SNI 主机名 (`SSL_ctrl(SSL_CTRL_SET_TLSEXT_HOSTNAME)`)
- 绑定 fd 并循环 `SSL_connect`
- 处理 `SSL_ERROR_WANT_READ/WRITE` 重试和 `SSL_ERROR_SYSCALL`

### Read / Write 实现
- `SSL_read` / `SSL_write` 包装
- 错误映射：`SSL_ERROR_WANT_READ/WRITE` → `WouldBlock`；`SSL_ERROR_SYSCALL` → `last_os_error()`

### Drop
- `SSL_shutdown` + `SSL_free` + `SSL_CTX_free` + TCP 关闭

## SocketStream (enum)

- `Plain(TcpStream)` — 非 TLS
- `Tls(OpenSslStream)` — OpenSSL 加密

### 方法
- `connect()`, `into_tls()`, `set_read_timeout()`, `set_write_timeout()`, `shutdown()`
- 实现 `Read` 和 `Write`
