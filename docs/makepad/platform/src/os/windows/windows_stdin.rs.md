# windows_stdin.rs — Studio 远程协议 stdin 事件循环

**文件路径**: platform/src/os/windows/windows_stdin.rs (390行)
**核心用途**: 实现 Makepad Studio 远程协议的 stdin 模式事件循环，通过 WebSocket 接收 Studio 指令，使用 D3D11 渲染并通过共享纹理交换链输出。

## 结构体

### `LocalPresentableImage`
```rust
struct LocalPresentableImage {
    id: PresentableImageId,
    image: Texture,
}
```

### `LocalSwapchain`
```rust
struct LocalSwapchain {
    presentable_images: [LocalPresentableImage; SWAPCHAIN_IMAGE_COUNT],
}
```

### `StdinWindow`
```rust
pub(crate) struct StdinWindow {
    swapchain: Option<LocalSwapchain>,
    present_index: usize,
    new_frame_being_rendered: Option<PresentableDraw>,
}
```

## 关键方法

### `Cx::stdin_event_loop`
主事件循环流程：
1. 发送 `BeforeStartup` 到 Studio 主机
2. 调用 `Event::Startup`，发送 `AfterStartup`
3. 循环处理 WebSocket 消息：
   - `Binary`: 反序列化为 `StudioToAppVec`，批量处理
   - `String`: 反序列化单个 `StudioToApp`
   - 每次消息处理完毕后调用 `handle_actions()` 和 `run_live_edit`

### `stdin_handle_host_to_stdin`

处理来自 Studio 的消息：

| StudioToApp 消息 | 处理 |
|-----------------|------|
| `MouseDown` | 根据坐标解析 window_id，调用 `dispatch_studio_msg` |
| `MouseMove` | 按鼠标按下窗口或坐标解析 window_id |
| `MouseUp` | 按鼠标按下窗口或坐标解析 window_id |
| `Scroll` | 根据坐标解析 window_id |
| `TweakRay` | 构造 `TweakRayEvent`，包含 DPI 因子、命中检测 |
| `WindowGeomChange` | 更新窗口几何，触发 `redraw_all` |
| `Swapchain` | 创建本地交换链，从共享句柄创建 D3D11 纹理，触发重绘 |
| `Tick` | 信号处理（终止、媒体、脚本、action_receiver）+ 网络事件 + `handle_platform_ops` + GC + `stdin_handle_repaint` + GPU 就绪检查 |
| 其他 | `dispatch_studio_msg` 统一分发 |

### `stdin_handle_platform_ops`

处理平台操作（简化版）：
- `CreateWindow` / `CreatePopupWindow`: 扩展 `stdin_windows` 数组，发送 `CreateWindow` 给 Studio 主机
- `SetCursor`: 转发 `AppToStudio::SetCursor`
- `CopyToClipboard`: 转发 `AppToStudio::SetClipboard`
- 其他操作：忽略（窗口管理在远程模式下由 Studio 控制）

### `stdin_handle_repaint`

渲染到共享纹理交换链：
1. `compute_pass_repaint_order` 确定渲染顺序
2. 对每个窗口 pass，在 `LocalSwapchain` 的当前图像上渲染
3. 调用 `draw_pass_to_texture`（输出到共享纹理）
4. 创建 `PresentableDraw`，包含窗口 ID、目标 ID、尺寸
5. 启动 GPU 查询，标记 `new_frame_being_rendered`
6. 下次 `Tick` 检查 GPU 就绪，发送 `DrawCompleteAndFlip`

### GPU 就绪检测

在 `Tick` 处理中：
```rust
if has_pending_draws && d3d11_cx.is_gpu_done() {
    for window in stdin_windows {
        if let Some(presentable_draw) = window.new_frame_being_rendered.take() {
            Self::stdin_send_to_host(AppToStudio::DrawCompleteAndFlip(presentable_draw));
        }
    }
}
```

### GC（垃圾回收）

在 `Tick` 中集成 Makepad 脚本 VM 的 GC：
1. 调用 `vm.heap().needs_gc()` 判断是否需要 GC
2. 执行 `vm.gc()`
3. 发送 `GCSample` 给 Studio 主机（包含开始时间、结束时间、堆存活对象数）

## 共享纹理机制

- Studio 主机分配共享纹理（`SharedBGRAu8`）并通过 `Swapchain` 消息发送
- 客户端通过 `Texture::update_from_shared_handle` 绑定到 D3D11 设备
- 渲染完成后通过 `DrawCompleteAndFlip` 通知主机切换显示

## 平台集成

- 由 `windows.rs` 的 `event_loop` 在检测到 `--stdin-loop` 模式时调用
- 使用 D3D11 渲染（共享 `D3d11Cx`）
- 协议序列化基于 `makepad_studio_protocol` crate
- 非 stdin 模式使用传统 Win32 窗口事件循环
