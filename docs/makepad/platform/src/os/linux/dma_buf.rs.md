# dma_buf.rs — Linux DMA-BUF 跨进程纹理共享

**文件路径**: `platform/src/os/linux/dma_buf.rs` (318 行)
**核心功能**: 定义通过 Linux DMA-BUF 机制在进程间共享 GPU 纹理的数据结构，支持序列化/反序列化。

## 背景

DMA-BUF 是 Linux 内核的机制，允许在不同进程/驱动间共享 GPU 缓冲区。此模块定义了在 EGL 环境下用于跨进程纹理共享的类型：
1. 进程 A 从 OpenGL 纹理创建 `EGLImage` 并导出 DMA-BUF
2. 进程 A 将文件描述符和元数据传递给进程 B
3. 进程 B 导入 DMA-BUF 作为 `EGLImage` 并绑定为 OpenGL 纹理

## 主要类型

### `Image<FD>`
包含 DRM 格式信息和图像平面（plane）的泛型结构体：
- `drm_format: DrmFormat` — DRM 图像格式
- `planes: ImagePlane<FD>` — 图像平面（当前只支持单平面 RGBA）

### `DrmFormat`
DRM 格式描述：
- `fourcc: u32` — FourCC 编码（如 `DRM_FORMAT_ABGR8888` = `0x34324241`）
- `modifiers: u64` — 格式修饰符（描述 tiling 等 GPU 布局信息）

### `ImagePlane<FD>`
单个 DMA-BUF 平面的描述：
- `dma_buf_fd: FD` — DMA-BUF 文件描述符
- `offset: u32` — 缓冲区起始偏移量
- `stride: u32` — 每行像素跨度（pitch）

## 序列化

所有类型都实现了 `SerBin` / `DeBin`（二进制）和 `SerJson` / `DeJson`（JSON）序列化，通过 `makepad_micro_serde` 宏实现。这使得 `Image<FD>` 可以在进程间通过 IPC 传递。

## 关键方法

- `Image::planes_fd_map(f)` — 将文件描述符类型 `FD` 映射为 `FD2`，用于 FD 传递前的类型转换
