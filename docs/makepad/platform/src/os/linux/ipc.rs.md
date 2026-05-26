# ipc.rs — 基于 Unix 域套接字的进程间通信

**文件路径**: `platform/src/os/linux/ipc.rs` (468 行)
**核心功能**: 通过 Unix 域套接字和 `sendmsg`/`recvmsg` 实现双向 IPC 通道，支持同时传输字节数据和文件描述符。

## 主要类型

### `Channel<TX, RX>`
双向 IPC 通道端点，使用 `UnixStream` 作为底层传输：
- `send` — 编码并发送类型 `TX` 的消息（包含字节和 FD）
- `recv` — 接收并解码类型 `RX` 的消息
- `into_child_process_inheritable` — 转换为子进程可继承的通道

### `InheritableChannel<TX, RX>`
`Channel` 的包装，其文件描述符去除了 `CLOEXEC` 标志，可在 `fork`/`exec` 后存活。支持 `AsFd` 转换和从 `OwnedFd` 构造。

### `Never`
空枚举，用于标记不可用的通道方向。

### `FixedSizeEncoding<BYTE_LEN, FD_LEN>`
将消息编码为固定大小"数据包"的 trait：
- `encode` — 将消息编码为 `[u8; BYTE_LEN]` 字节数组和 `[BorrowedFd; FD_LEN]` 文件描述符数组
- `decode` — 从字节数组和文件描述符解码消息

## 底层实现

### `sys` 模块（两个实现）

**稳定路径**（默认）：通过 `libc` 的 `sendmsg`/`recvmsg` FFI 直接调用，手动构造 `msghdr` 和 `cmsghdr` 结构体。

**不稳定路径**（`cfg(use_unstable_unix_socket_ancillary_data_2021)`）：使用 Rust 标准库实验性的 `SocketAncillary` API。

### `stream_sendmsg` / `stream_recvmsg`
- 使用 `SCM_RIGHTS` 辅助数据传递文件描述符
- 发送：构造 `CMsgBuf` 包含 `cmsghdr` 和 FD 数组
- 接收：验证 `cmsg_len`、`cmsg_level`、`cmsg_type` 完整性
- 错误处理：检查部分读写、辅助缓冲区截断、无效 FD

## 编码器实现

### `FixedSizeEncoding` 的派生实现
- `u16`/`u32`/`u64`/`u128` — 小端字节序编码，无 FD
- `OwnedFd` — 空字节，单一 FD
- `(A, B)` 元组 — 组合字节和 FD 编码（`A` 编码字节，`B` 编码 FD）
