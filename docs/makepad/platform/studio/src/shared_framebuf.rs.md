# shared_framebuf.rs — 共享帧缓冲与交换链

## 概述

`shared_framebuf.rs` 定义了跨进程共享图形内存所需的数据结构，支持 Makepad Studio 的运行预览视图（RunView）将应用渲染结果从子进程传输到 Studio 进程进行显示。包含交换链配置、图像标识、平台特定共享图像描述以及绘制完成通知。

---

## SWAPCHAIN_IMAGE_COUNT

```rust
pub const SWAPCHAIN_IMAGE_COUNT: usize = match () {
    _ if cfg!(target_os = "linux") => 3,   // Linux: 三重缓冲
    _ if cfg!(target_os = "macos") => 1,   // macOS: 单缓冲
    _ if cfg!(target_os = "windows") => 2, // Windows: 双重缓冲
    _ => 2,                                 // 其他: 双重缓冲
};
```

为不同平台优化交换链深度：
- **Linux**: 3 帧（三重缓冲，配合 DMA-BUF 的零拷贝管线）
- **macOS**: 1 帧（IOSurface 单缓冲，系统管理呈现）
- **Windows**: 2 帧（传统双重缓冲）
- **其他/OHOS**: 2 帧

---

## PresentableImageId — 可呈现图像唯一标识

```rust
#[derive(Copy, Clone, Debug, PartialEq, SerBin, DeBin, SerJson, DeJson)]
pub struct PresentableImageId {
    origin_pid: u32,            // 创建进程 PID
    per_origin_counter: u32,    // 进程内自增计数器
}
```

### 方法

```rust
pub fn alloc() -> Self                           // 分配新 ID（进程 PID + 原子计数器）
pub fn as_u64(self) -> u64                       // 压缩为 u64 (PID << 32 | counter)
pub fn from_u64(pid_and_counter: u64) -> Self    // 从 u64 还原
```

`alloc()` 使用 `AtomicU32` 作为全局计数器，确保同一进程内分配的 ID 唯一。

---

## DrmFormat — Linux DRM 格式

```rust
#[cfg(all(target_os = "linux", not(target_env = "ohos")))]
pub struct DrmFormat {
    pub fourcc: u32,     // DRM FOURCC 格式码（如 DRM_FORMAT_ARGB8888）
    pub modifiers: u64,  // 修饰符（tiling/压缩模式）
}
```

仅 Linux（非 OHOS）可用。

---

## Linux 共享图像

```rust
// AuxChannedImageFd — 辅助通道文件描述符包装
pub struct AuxChannedImageFd { pub _private: Option<u32> }

// LinuxSharedImagePlane — DMA-BUF 图像平面
pub struct LinuxSharedImagePlane {
    pub dma_buf_fd: AuxChannedImageFd,  // DMA-BUF 文件描述符
    pub offset: u32,                     // 平面偏移
    pub stride: u32,                     // 行跨度（字节）
}

// LinuxSharedImage — DMA-BUF 图像
pub struct LinuxSharedImage {
    pub drm_format: DrmFormat,          // DRM 格式描述
    pub plane: LinuxSharedImagePlane,   // 平面描述
}

// Linux SharedPresentableImage
pub struct SharedPresentableImage {
    pub id: PresentableImageId,         // 唯一 ID
    pub image: LinuxSharedImage,        // DMA-BUF 图像
}
```

---

## macOS 共享图像

```rust
#[cfg(target_os = "macos")]
pub struct SharedPresentableImage {
    pub id: PresentableImageId,     // 唯一 ID
    pub iosurface_id: u32,           // IOSurface ID（系统级共享内存标识）
}
```

---

## Windows 共享图像

```rust
#[cfg(target_os = "windows")]
pub struct SharedPresentableImage {
    pub id: PresentableImageId,     // 唯一 ID
    pub handle: u64,                // NT 句柄（跨进程共享 GPU 资源）
}
```

---

## 回退实现

当平台不是 Linux、macOS 或 Windows 时（例如 Android、OHOS 或测试环境）：

```rust
#[cfg(not(any(linux, macos, windows)))]
pub struct SharedPresentableImage {
    pub id: PresentableImageId,
    pub _dummy: Option<u32>,
}
```
`_dummy` 字段保持结构体在所有平台上具有相似的字段数量。

---

## SharedSwapchain — 共享交换链

```rust
pub struct SharedSwapchain {
    pub window_id: usize,                                              // 窗口标识
    pub alloc_width: u32,                                              // 分配宽度
    pub alloc_height: u32,                                             // 分配高度
    pub presentable_images: [SharedPresentableImage; SWAPCHAIN_IMAGE_COUNT], // 图像池
}
```

`window_id` 对应 Studio 中的 RunView 窗口。交换链在 Studio 端分配，通过 `StudioToApp::Swapchain` 传递给应用进程。

---

## PresentableDraw — 绘制完成通知

```rust
/// 客户端完成绘制后，Host 只需知道以下信息即可呈现结果：
/// - 使用的交换链图像
/// - 绘制覆盖的子区域（当前为整个窗口）
pub struct PresentableDraw {
    pub window_id: usize,                // 窗口标识
    pub target_id: PresentableImageId,   // 渲染目标图像 ID
    pub width: u32,                      // 绘制区域宽度
    pub height: u32,                     // 绘制区域高度
}
```

通过 `AppToStudio::DrawCompleteAndFlip(PresentableDraw)` 从应用进程发送到 Studio，通知 Studio 可以呈现该帧。

---

## 跨进程数据流

```
Studio 进程                        应用子进程
    │                                   │
    │  StudioToApp::Swapchain ─────────>│  发送交换链描述
    │                                   │  应用渲染到共享图像
    │  <─── AppToStudio::DrawCompleteAndFlip  通知绘制完成
    │                                   │
    │  呈现共享图像到 RunView           │
    │                                   │
```

在不同平台上，共享机制不同：
- **Linux**: DMA-BUF 文件描述符通过 Unix domain socket 传递
- **macOS**: IOSurface ID 跨进程共享
- **Windows**: NT Handle 跨进程共享 GPU 资源
