# linux_x11_stdin.rs — Studio stdin-loop Mode for X11

**文件路径**: platform/src/os/linux/x11/linux_x11_stdin.rs (612行)
**核心用途**: 实现 Makepad Studio 的 stdin-loop 模式，在此模式下应用作为 Studio 的子进程运行，通过 WebSocket 和辅助通道与 Studio 主机通信。

## Types/Structs

### `StdinWindow`
stdin-loop 模式下的窗口状态。

| 字段 | 类型 | 描述 |
|------|------|------|
| `swapchain` | `Option<HostSwapchain>` | Studio 主机的 swapchain（渲染目标） |
| `present_index` | `usize` | 当前展示的图像索引 |
| `readback_framebuffer` | `Option<u32>` | 软件回退的 GL 帧缓冲（glReadPixels） |
| `last_trace_draw` | `Option<(u32,u32,u32,u32,u64)>` | 最近的 DPI 跟踪信息 |

## Key Methods

### `Cx::stdin_handle_repaint(windows)`
在 stdin-loop 模式下的重绘实现：
1. 获取 OpenGL 上下文
2. 计算 pass 重绘顺序
3. 对每个窗口 pass：
   - 渲染到 swapchain 的当前可呈现图像
   - `glFinish` 等待 GPU 完成
   - 若有软件缓冲，通过 `glReadPixels` 读取像素
   - 编码 width/height 信息到缓冲区
   - 发送 `AppToStudio::DrawCompleteAndFlip`
4. 对纹理 pass 直接渲染到纹理

### `Cx::stdin_event_loop()`
stdin-loop 主循环（替代 x11_event_loop）：
1. 发送 `BeforeStartup`，处理 Startup，发送 `AfterStartup`
2. 处理初始 platform_ops
3. 循环接收 WebSocket 消息（`StudioToAppVec` 二进制或 `StudioToApp` JSON）
4. 分发 `StudioToApp` 消息处理

### `Cx::stdin_handle_host_to_stdin(msg, aux_chan, stdin_windows) -> bool`
处理来自 Studio 主机的消息：

| StudioToApp 变体 | 处理方式 |
|------------------|---------|
| `MouseDown/Move/Up/Scroll` | 根据坐标解析 window_id，委托给 `dispatch_studio_msg` |
| `TweakRay` | 处理调试射线事件 |
| `WindowGeomChange` | 更新窗口几何，触发重绘 |
| `Swapchain` | 接收新 swapchain（DMA-BUF 或软件回退），设置到窗口 |
| `Tick` | 信号处理、定时器、网络、重绘、垃圾回收 |
| 其他 | 通过 `dispatch_studio_msg` 通用处理 |

### `Cx::stdin_handle_platform_ops(stdin_windows)`
处理 stdin-loop 下的平台操作：
- `CreateWindow`：扩展 stdin_windows 向量，通知主机
- `CreatePopupWindow`：扩展向量
- `SetCursor`：通知主机
- `StartTimer/StopTimer`：使用 `PollTimer` 管理
- `CopyToClipboard`：通知主机设置剪贴板
- `HttpRequest/CancelHttpRequest`：网络请求

### 辅助函数
- `stdin_send_to_host(msg)` — 发送 `AppToStudio` 消息到主机
- `stdin_aux_chan_endpoint(endpoint)` — 连接或返回辅助通道端点

## Implementation Details
- 软件回退：当共享内存不可用时，通过 `glReadPixels` 读取渲染结果
- 尺寸编码：在缓冲区的前8字节写入 width/height（包含在首行和末行）
- 垃圾回收：每个 Tick 检查 VM 堆是否需要 GC，生成 `GCSample` 报告
- swapchain 颜色纹理使用 `SharedBGRAu8` 或 `RenderBGRAu8`（软件回退）
