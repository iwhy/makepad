# `shared_bytes.rs` — 引用计数字节缓冲（模块转发）

## 概述

该文件仅为公共 API 的转发（re-export）点，所有实现位于独立的 `makepad_shared_bytes` crate 中（`libs/shared_bytes/src/lib.rs`）。文件中导出了三个类型。

## 转发类型

### `SharedBytes`
引用计数字节缓冲枚举，包含两个变体：
- **`Owned(Rc<Vec<u8>>)`**：持有 `Rc<Vec<u8>>` 引用计数字节向量。多个消费者可以共享同一份堆数据，仅在所有引用释放后才回收。
- **`Mapped(Rc<MappedBytes>`**：持有内存映射文件的引用计数包装。文件数据通过操作系统 MMAP 机制按需分页加载，支持大文件的高效随机访问。

创建方法包括：
- `from_owned(Rc<Vec<u8>>)` / `from_vec(Vec<u8>)`：从所有权字节构建。
- `from_file(path)`：读取文件全部内容到内存（fallback 路径）。
- `from_file_mmap_or_read(path)`：优先尝试 MMAP，失败时回退为常规读取。通过 `MMAP_HITS` / `MMAP_FALLBACKS` 原子计数器追踪命中率。

`as_slice()` 提供统一的无拷贝字节切片访问，无论底层是堆分配还是 MMAP。`stats()` / `reset_stats()` 全局统计接口用于诊断 I/O 策略效率。

### `MappedBytes`
内存映射文件的平台无关包装。内部持有 `MappedBytesInner`，在不同平台上有不同实现：
- **Unix (Linux/macOS)**：通过 `mmap` / `munmap` 系统调用实现，使用 `MAP_PRIVATE | PROT_READ` 标记。
- **Windows**：通过 `CreateFileMappingW` / `MapViewOfFile` / `UnmapViewOfFile` kernel32 API 实现。
- **其他平台（包括 wasm32）**：返回 `Unsupported` 错误。

`Drop` 实现自动释放映射区域，确保资源安全。

### `SharedBytesStats`
统计结构，包含 `mmap_hits`（MMAP 成功数）、`mmap_fallbacks`（MMAP 失败回退数）、`owned_loads`（常规加载数），使用 `AtomicU64` 保证线程安全。
