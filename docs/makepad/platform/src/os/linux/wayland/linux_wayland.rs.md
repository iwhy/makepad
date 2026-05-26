# linux_wayland.rs — Main Wayland Backend Implementation

**文件路径**: platform/src/os/linux/wayland/linux_wayland.rs (1134行)
**核心用途**: 实现 Wayland 后端的核心事件循环，处理平台操作(CxOsOp)分发、窗口生命周期管理、视频播放管道以及重绘调度。

## Types/Structs

### `WaylandCx`
Wayland 上下文，连接 Wayland 事件循环与 Makepad 的 Cx 系统。

| 字段 | 类型 | 描述 |
|------|------|------|
| `cx` | `Rc<RefCell<Cx>>` | 共享的 Makepad 核心上下文 |
| `qhandle` | `Option<wayland_client::QueueHandle<WaylandState>>` | Wayland 事件队列句柄 |

## Key Methods/Functions

### `wayland_event_loop(cx: Rc<RefCell<Cx>>)`
Wayland 事件循环的公共入口点。委托给 `WaylandCx::event_loop_impl`。

### `WaylandCx::event_loop_impl(cx: Rc<RefCell<Cx>>)`
初始化 Wayland 渲染后端：
1. 设置 `OsType::LinuxWindow` 和 GPU 性能层级
2. 连接 Wayland 服务器，创建 EGL 平台显示
3. 检查 `--stdin-loop` 模式（Makepad Studio 子进程渲染）
4. 创建事件队列，获取 registry，初始化 `WaylandState`
5. 创建 `WaylandApp`，发送 `Event::Startup`
6. 启动 8ms 定时器，进入事件循环

### `WaylandCx::state_event_callback(state, event) -> EventFlow`
处理来自 WaylandState 的回调事件。逐一匹配 `XlibEvent` 变体：

- **WindowGotFocus**: 重绘所有窗口和弹出窗口
- **WindowLostFocus**: 转发失焦事件
- **WindowGeomChange**: 更新窗口几何、DPI，处理自定义窗口按钮区域(chrome buttons)
- **WindowClosed**: 清理窗口/弹出窗口，关闭子弹出窗口，退出条件判断
- **PopupDismissed**: 转发弹窗关闭事件
- **Paint**: 调用 `call_next_frame_event`、`call_draw_event`、编译着色器、触发 `handle_repaint`
- **MouseDown/MouseUp/Move/Scroll/Drag/Drop/DragEnd**: DPI 缩放后转发事件
- **KeyDown/KeyUp**: 通过 `cx.keyboard` 处理按键事件
- **TextCopy/TextCut**: 转发剪贴板事件
- **Timer**: timer_id=0 时处理信号、控制通道、action、网络、视频播放轮询；其他 id 处理脚本定时器

### `WaylandCx::app_event_callback(wayland_app, event) -> EventFlow`
WaylandApp 层回调，委托给 `state_event_callback`，并在需要时终止事件循环。

### `WaylandCx::close_popup_window(state, window_id, reason)`
关闭弹出窗口：发送 PopupDismissed 和 WindowClosed 事件，清理指针/键盘窗口。

### `WaylandCx::close_popup_children(state, parent_window_id)`
递归关闭指定窗口的所有子弹出窗口。

### `WaylandCx::handle_platform_ops(state) -> EventFlow`
处理 `cx.platform_ops` 队列中的所有 CxOsOp：

| CxOsOp 变体 | 处理方式 |
|-------------|---------|
| `CreateWindow` | 创建 WaylandWindow，设置 xdg-toplevel、装饰管理器、viewporter、分數縮放 |
| `CreatePopupWindow` | 创建 WaylandPopupWindow |
| `CloseWindow` | 递归关闭子弹出窗口 |
| `Quit` | 设置退出标志 |
| `MinimizeWindow` | 调用 `toplevel.set_minimized()` |
| `MaximizeWindow` | 调用 `toplevel.set_maximized()` |
| `FullscreenWindow` | 调用 `toplevel.set_fullscreen(None)` |
| `RestoreWindow` | 取消最大化/全屏 |
| `CopyToClipboard` | 通过 data_device 设置剪贴板文本 |
| `SetPrimarySelection` | 通过 primary_selection 协议设置主选择 |
| `StartDragging` | 启动内部拖拽 |
| `SetCursor` | 通过 cursor_shape 协议设置光标形状 |
| `StartTimer/StopTimer` | 更新 SelectTimers |
| `HttpRequest/CancelHttpRequest` | HTTP 网络请求 |
| `ShowTextIME/HideTextIME` | 启用/禁用 text_input 协议 |
| `PrepareVideoPlayback` | 视频/摄像头播放初始化（V4L2/GStreamer/Software 回退） |
| `Begin/Pause/Resume/Mute/Unmute/Seek/Volume/Rate` | 视频播放控制 |
| `CleanupVideoPlaybackResources` | 清理视频播放资源 |
| `PrepareAudioPlayback` | 音频播放初始化（GStreamer） |
| `CheckPermission/RequestPermission` | Linux 桌面默认授予所有权限 |

### `WaylandCx::handle_repaint(state)`
重绘调度：计算 pass 重绘顺序，对每个 Window pass：
- 检查窗口是否已配置
- 调用 `resize_buffers`
- 应用 viewport 缩放（若支持）
- 计算物理像素尺寸，调用 `cx.draw_pass_to_window`
弹出窗口和纹理 pass 有类似的处理路径。

## Implementation Details
- 视频播放有三种后端尝试：V4L2 摄像头 > GStreamer > 软件解码(rav1d)
- 环境变量 `MAKEPAD_FORCE_SOFTWARE_VIDEO` 强制使用软件解码
- Window backdrop 在 Wayland 下不支持（无操作日志）
- 滚动事件：使用 `XlibEvent::Scroll` 统一抽象，同时支持鼠标滚轮和触摸板
- 键盘重复：通过 `state.keyboard_serial` 和剪贴板冲刷保证时序

## Platform Integration
通过 `egl_sys::EGL_PLATFORM_WAYLAND_KHR` 初始化 EGL，绑定到 Wayland 显示后端。Wayland 窗口通过 wl_surface + xdg_surface/xdg_toplevel 实现，EGL 窗口表面通过 `WlEglSurface` 创建。
