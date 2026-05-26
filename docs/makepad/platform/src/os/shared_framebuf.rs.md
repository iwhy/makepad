# `shared_framebuf.rs` — 共享帧缓冲协议（Studio 远程显示）

## 文件定位

此文件实现 Makepad Studio 远程显示功能所需的帧缓冲共享机制。它定义了应用端（Host）持有的可展示图像和交换链结构，并提供了按平台序列化为 `SharedSwapchain`（通过 IPC 发送给 Studio 前端）的函数。

核心设计模式是**通过平台特定的共享内存机制（IOSurface、DMA-BUF、HANDLE）将 GPU 纹理直接暴露给 Studio 进程**，避免像素数据的 CPU 回读。

---

## 一、辅助函数

### `ref_array_to_array_of_refs<T, const N: usize>(ref_array: &[T; N]) -> [&T; N]`

- **实现逻辑**：
  1. 使用 `MaybeUninit<[&T; N]>` 创建未初始化的引用数组。
  2. 遍历输入数组的引用，通过指针算术将它们逐个写入输出数组的对应位置。
  3. 使用 `unsafe { out_refs.assume_init() }` 标记初始化完成。
- **用途**：将 `[T; N]` 转为 `[&T; N]`，弥补 Rust 标准库中 `[T]::each_ref` 尚未稳定化的空白。

---

## 二、Host 端交换链与应用端结构

### `HostPresentableImage`

```rust
pub struct HostPresentableImage {
    pub id: PresentableImageId,
    pub texture: Texture,
    #[cfg(all(target_os = "linux", not(target_env = "ohos")))]
    pub software_buffer: Option<LinuxSharedSoftwareBuffer>,
}
```

- **职责**：应用端持有的单个可展示图像。
- **字段说明**：
  - `id`：全局唯一标识，用于在 IPC 消息中引用此图像。
  - `texture`：底层的 GPU 纹理对象。
  - `software_buffer`（Linux 特有）：可选的软件回退缓冲区，当 DMA-BUF 导出失败时使用。

### `HostSwapchain`

```rust
pub struct HostSwapchain {
    pub window_id: usize,
    pub alloc_width: u32,
    pub alloc_height: u32,
    pub presentable_images: [HostPresentableImage; SWAPCHAIN_IMAGE_COUNT],
}
```

- **职责**：应用端持有的交换链，包含固定数量的可展示图像（通常为 3 个，实现三重缓冲）。
- **窗口 ID** 用于关联到特定的 Studio RunView。

### `HostSwapchain::new(window_id, alloc_width, alloc_height, cx)`

- **实现逻辑**：
  1. 使用 `std::array::from_fn` 创建 `SWAPCHAIN_IMAGE_COUNT` 个图像。
  2. 每个图像调用 `PresentableImageId::alloc()` 分配唯一 ID。
  3. 创建共享格式纹理 `TextureFormat::SharedBGRAu8`，传入 ID、宽、高和 `initial: true`（预先分配后备存储）。
  4. Linux 上初始 `software_buffer` 为 `None`。

### `HostSwapchain::get_image(&self, id: PresentableImageId) -> Option<&HostPresentableImage>`

- **实现逻辑**：
  1. 使用迭代器的 `find` 方法，在 `presentable_images` 中查找 `pi.id == id`。
  2. 找到返回 `Some(&image)`，未找到返回 `None`。

### `HostSwapchain::regenerate_ids(&mut self)`

- **实现逻辑**：
  1. 遍历所有 `presentable_images`。
  2. 为每个图像重新分配 `PresentableImageId`。
- **用途**：当交换链需要重新同步时（例如 Studio 重连），更新所有图像 ID 使旧引用失效。

---

## 三、`shared_swapchain_from_host_swapchain` — 平台序列化

此函数有三个平台特定实现和一个 fallback。

### macOS 实现：IOSurface

```rust
#[cfg(target_os = "macos")]
pub fn shared_swapchain_from_host_swapchain(host: &HostSwapchain, cx: &mut Cx) -> SharedSwapchain
```

- **实现逻辑**：
  1. 构造 `SharedSwapchain`，复制 `window_id`、`alloc_width`、`alloc_height`。
  2. 对每个 `HostPresentableImage`，通过 `cx.share_texture_for_presentable_image(&texture)` 获取 IOSurface ID。
  3. IOSurface ID 是一个 `mach_port_t`，可以在不同进程间传递，Studio 进程通过它直接访问 GPU 纹理。

### Windows 实现：HANDLE

```rust
#[cfg(target_os = "windows")]
pub fn shared_swapchain_from_host_swapchain(host: &HostSwapchain, cx: &mut Cx) -> SharedSwapchain
```

- **实现逻辑**：
  1. 与 macOS 类似，但调用 `cx.share_texture_for_presentable_image` 返回的是 Windows HANDLE（适配于 `ID3D11Device::OpenSharedResource` 或 `ID3D12Device::CreateSharedHandle`）。
  2. HANDLE 可以跨进程共享，Studio 通过它打开同一 GPU 资源。

### Linux 实现：DMA-BUF + Aux Channel

```rust
#[cfg(all(target_os = "linux", not(target_env = "ohos")))]
pub fn shared_swapchain_from_host_swapchain(
    host: &mut HostSwapchain,  // 注意：&mut
    cx: &mut Cx,
    host_endpoint: &aux_chan::HostEndpoint,  // 注意：额外参数
) -> Result<SharedSwapchain, SharedSwapchainCreateError>
```

- **实现逻辑**：
  1. 首先尝试通过 `cx.share_texture_for_presentable_image` 为每个图像导出 DMA-BUF fd。
  2. 如果任意一个图像导出失败（返回 `None`），启用软件回退模式。
  3. **DMA-BUF 模式**：所有图像导出成功时，清空 `software_buffer`，通过 `aux_chan::send_image_fds_to_aux_chan` 将 DMA-BUF fd 和 DRM 格式信息通过辅助通道发送给 Studio。
  4. **软件回退模式**：首次回退时打印 warning。对每个图像调用 `software_fallback_image` 创建/获取 `LinuxSharedSoftwareBuffer`（内存映射文件），克隆 fd 通过辅助通道发送。Studio 端通过 mmap 读取像素数据。
  5. 构造 `SharedSwapchain` 返回。
- **设计要点**：
  - DMA-BUF 路径 Zero-Copy：GPU 直接生成帧数据，fd 跨进程传递，不涉及 CPU。
  - 软件回退路径 One-Copy：GPU → CPU readback → memfd → Studio mmap。
  - `host: &mut HostSwapchain` 是因为软件回退需要修改 `host.presentable_images[i].software_buffer`。

### Fallback 实现

```rust
#[cfg(not(any(target_os = "linux", target_os = "macos", target_os = "windows")))]
pub fn shared_swapchain_from_host_swapchain(...)
```

- 返回一个 `_dummy: None` 的 `SharedSwapchain`，无法在 Studio 中远程显示。

---

## 四、Linux 软件回退缓冲区

### `LinuxSharedSoftwareBuffer`

- **成员**：`fd`（memfd 文件描述符）、`ptr`（mmap 指针）、`len`（大小）、`stride`（行跨度）。

### `create(len, stride)`

- **实现逻辑**：
  1. 调用 `memfd_create("makepad-runview", MFD_CLOEXEC)` 创建匿名内存文件。
  2. 调用 `ftruncate` 将文件大小设为所需长度。
  3. 调用 `mmap` 将文件映射到进程地址空间，使用 `PROT_READ | PROT_WRITE` 和 `MAP_SHARED`。
  4. 返回包含 fd、指针和步长的结构。

### `from_fd(fd, len, stride)`

- **实现逻辑**：从已有的 fd 创建（接收端），直接 `mmap` 共享内存，不创建新文件。

### `clone_fd() -> OwnedFd`

- 使用 `try_clone_to_owned` 复制 fd，便于将同一缓冲区共享给 Studio。

---

## 五、Linux Aux 通道（`aux_chan` 模块）

辅助通道是在标准输入/输出之外的独立 Unix 域套接字，用于在应用进程和 Studio 进程间传输 DMA-BUF fd 和 DRM 格式元数据。

### `path_for_studio(studio_host, studio_build_id) -> PathBuf`

- **实现逻辑**：
  1. 从 `STUDIO_HOST` 中提取 host:port（去掉 `ws://` 前缀和路径部分）。
  2. 从参数或 `resolve_studio_build` 获取 build ID。
  3. 构造 socket 路径：`/tmp/makepad-stdin-aux-{port}-{build_id}.sock`。
- **设计要点**：路径包含端口和 build ID，允许多个 Makepad 实例在同一台机器上并行运行而不冲突。

### `ExternalEndpointListener`

负责监听 Unix 域套接字，等待 Studio 连接。

- **`new_for_studio(studio, studio_build_id)`**：
  1. 计算 socket 路径。
  2. 尝试 `remove_file` 清理之前的 socket 文件（如果存在）。
  3. 创建 `UnixListener` 并设为非阻塞模式。

- **`accept_host_endpoint(&self) -> io::Result<HostEndpoint>`**：
  1. 设置 120 秒超时。
  2. 轮询 `self.listener.accept()`，如果返回 `WouldBlock` 或 `Interrupted`，休眠 5ms 后重试。
  3. 连接成功后，将 `UnixStream` 包装为 `InheritableChannel`，再转为非可继承的 `Channel` 返回。

### `ClientEndpoint`

- **`connect_from_studio_env()`**：
  1. 从环境变量获取 Studio host。
  2. 计算 socket 路径。
  3. 设置 10 秒超时。
  4. 尝试连接 `UnixStream::connect`，如果遇到 `NotFound` / `ConnectionRefused` / `Interrupted`，休眠 5ms 后重试。

### 类型别名

```rust
pub type H2C = (PresentableImageId, OwnedFd);  // Host → Client 消息
pub type C2H = linux_ipc::Never;               // Client → Host 无消息
```

- 通信方向是单向的：Host 将 DMA-BUF fd 发送给 Client（Studio）。

### `FixedSizeEncoding` for `PresentableImageId`

- **实现逻辑**：
  - `encode`：将 `as_u64()` 结果编码为字节数组，不附带任何 fd。
  - `decode`：从字节数组解码回 `PresentableImageId`。

### `send_image_fds_to_aux_chan(id, image, host_endpoint)`

- **实现逻辑**：
  1. 解构 `LinuxOwnedImage` 提取 DRM 格式和平面信息。
  2. 通过 `host_endpoint.send((id, plane.dma_buf_fd))` 发送 ID + fd 对。
  3. 构造 `LinuxSharedImage` 返回（包含 DRM 格式、偏移、步长和一个标记 `AuxChannedImageFd`）。

### `recv_image_fds_from_aux_chan(id, image, client_endpoint)`

- **实现逻辑**：
  1. 循环从 `client_endpoint.recv()` 接收 `(recv_id, recv_fd)` 对。
  2. 如果 `recv_id` 与请求的 ID 不匹配，计为一次失配并继续循环。
  3. 连续失配 64 次则返回错误（假设通道损坏或不同步）。
  4. 匹配成功时，如果之前有失配，记录 warning（丢弃了过期的交换链图像）。
  5. 返回 `LinuxOwnedImage`。

---

## 六、非 Linux 平台的 `aux_chan` 桩

```rust
#[cfg(not(all(target_os = "linux", not(target_env = "ohos"))))]
pub mod aux_chan { ... }
```

- 提供 `HostEndpoint`、`ClientEndpoint`、`ExternalEndpointListener` 的最小实现。
- `connect_from_studio_env` 和 `new_for_studio` 总是成功，返回空结构体。
- Linux 平台独占的辅助通道在其他平台上为空操作，确保交叉编译时模块存在。

---

## 七、`WindowKindId` — 窗口类型标识

```rust
#[repr(usize)]
pub enum WindowKindId {
    Main = 0,
    Design = 1,
    Outline = 2,
}
```

- **用途**：区分 Studio 远程工具中的不同窗口类型，用于关联截屏请求。
- `from_usize(d)` 将 `usize` 映射为枚举值，非法值 panic。

---

## 八、`PollTimer` 与 `PollTimers` — 轮询计时器

### `PollTimer`

```rust
pub struct PollTimer {
    pub start_time: Instant,
    pub interval: Duration,
    pub repeats: bool,
    pub step: u64,
}
```

- **职责**：保存单个计时器的状态。
- `start_time`：计时器创建时间。
- `interval`：触发间隔。
- `repeats`：是否重复触发。
- `step`：已触发的次数（用于计算下次到期时间）。

### `PollTimers`

```rust
pub struct PollTimers {
    pub timers: HashMap<u64, PollTimer>,
    pub time_start: Instant,
    pub last_time: Instant,
}
```

- **职责**：管理一组计时器，基于 `Instant` 而非系统时钟。
- `Default` 实现：初始化 `time_start` 和 `last_time` 为当前时刻。

### `PollTimers::time_now(&self) -> f64`

- **实现逻辑**：计算从 `time_start` 到当前时刻的秒数（浮点数）。
- 注释掉了 `mach_absolute_time`，使用 Rust 标准库替代。

### `PollTimers::get_dispatch(&mut self) -> Vec<TimerEvent>`

- **职责**：检查所有计时器，返回到期的计时器事件列表。
- **实现逻辑**：
  1. 计算当前时间 `now` 和相对于启动的时间 `time`。
  2. 遍历所有计时器，计算下次到期时间 `interval * (step + 1)`。
  3. 如果 `elapsed_time > next_due_time`，构造 `TimerEvent { timer_id, time }` 加入调度列表。
  4. 重复计时器 `step += 1`；一次性计时器加入待移除列表。
  5. 从 `self.timers` 中移除一次性的到期计时器。
  6. 更新 `last_time`。
  7. 返回 `TimerEvent` 列表。
- **用途**：在事件循环的轮询路径中调用，替代 `TimerQueue` 中的基于 `std::time::SystemTime` 的计时器。

---

## 九、总结

`shared_framebuf.rs` 专注于单个核心任务——**将应用端的 GPU 帧缓冲跨进程共享给 Studio**。它通过三条路径实现：

| 平台 | 共享机制 | 数据流 |
|------|---------|--------|
| macOS | IOSurface | GPU → IOSurface → Studio（Zero-Copy） |
| Windows | Shared HANDLE | GPU → Shared Resource → Studio（Zero-Copy） |
| Linux（主路径） | DMA-BUF + Aux Channel | GPU → dma_buf fd → Unix Socket → Studio（Zero-Copy） |
| Linux（回退） | memfd + mmap | GPU → CPU Readback → memfd → Studio（One-Copy） |

辅助的 `PollTimers` 提供了一种在无原生窗口系统（或在主事件循环的 polling 模式下）管理计时器的机制。
