# windows/socket_stream.rs — Windows WinRT SocketStream

**File path**: `platform/network/src/backend/windows/socket_stream.rs` (202 行)
**Core purpose**: 使用 Windows `StreamSocket` WinRT API 实现 TCP/TLS socket 流，通过 `DataReader`/`DataWriter` 执行 I/O。

## SocketStream

### 字段
- `socket: Option<StreamSocket>` — WinRT socket
- `reader: Option<DataReader>` — 输入读取器（`InputStreamOptions::Partial`）
- `writer: Option<DataWriter>` — 输出写入器

### connect(host, port, use_tls, ignore_ssl_cert)
- 创建 `HostName` 和 `StreamSocket`
- 启用 TCP_NODELAY
- `ignore_ssl_cert` + TLS → 添加可忽略的证书错误类型（`Untrusted`, `InvalidName`, `Expired`, `IncompleteChain`, `Revoked`）
- 异步连接：
  - TLS → `ConnectWithProtectionLevelAsync(Tls12)`
  - 纯 TCP → `ConnectAsync`
- 创建 `DataReader` / `DataWriter`

### into_tls()
- 返回 `Unsupported`（Windows socket stream 当前不支持 TLS 升级）

### set_read_timeout / set_write_timeout
- 当前为空操作（Windows 实现暂不支持）

### shutdown()
- 依次关闭 `Writer` → `Reader` → `Socket`（通过 `IClosable` 接口）

### Read 实现
- 通过 `DataReader::LoadAsync` + `ReadBytes` 读取
- 使用 `executor::block_on` 同步等待 async 操作

### Write 实现
- `DataWriter::WriteBytes` + `StoreAsync` + `FlushAsync`
