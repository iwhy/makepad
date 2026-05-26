# libc_sys.rs — 最小化 libc FFI 绑定

**文件路径**: `platform/src/os/linux/libc_sys.rs` (110 行)
**核心功能**: 框架所需的 Linux libc 函数和类型的精简 FFI 绑定。

## 类型定义

- `time_t` / `suseconds_t` / `off_t` / `size_t` — POSIX 基本类型
- `fd_set` — `select()` 系统调用的文件描述符集合
- `timeval` — 带微秒精度的时间结构体

## 常量

### 文件操作
- `EPIPE` (32) / `ESPIPE` (29) — 管道错误码
- `O_RDWR` / `O_NONBLOCK` — 打开文件标志
- `F_GETFL` / `F_SETFL` — fcntl 命令

### 内存映射
- `PROT_READ` / `PROT_WRITE` — mmap 保护标志
- `MAP_SHARED` / `MAP_PRIVATE` / `MAP_FAILED` — mmap 映射类型
- `MFD_CLOEXEC` — memfd_create 标志

### 动态链接
- `RTLD_LAZY` / `RTLD_LOCAL` — dlopen 标志

### 系统调用
- `SYS_GETTIT` (186 for x86_64 / 178 for aarch64)

## FFI 函数

### 动态链接库
`dlopen` / `dlsym` / `dlclose`

### 文件与内存
`open` / `close` / `read` / `write` / `fcntl` / `free` / `pipe` / `ftruncate`

### 选择 I/O
`select` — 使用 `fd_set` 的 I/O 多路复用

### 内存映射
`mmap` / `munmap` / `memfd_create`

### 系统调用
`syscall` — 通用系统调用入口

## 辅助函数

- `FD_SET(fd, set)` — 在 `fd_set` 中设置文件描述符
- `FD_ZERO(set)` — 清空 `fd_set`

## 平台适应

根据目标指针宽度（32/64 位）自动选择 `ULONG_SIZE`。

## 用途

此模块不是完整的 libc 绑定，仅包含 Makepad 框架在 Linux 平台运行时所需的最小函数子集。
