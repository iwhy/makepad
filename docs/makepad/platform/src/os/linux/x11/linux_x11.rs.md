# linux_x11.rs — Main X11 Backend Implementation

**文件路径**: platform/src/os/linux/x11/linux_x11.rs (1015行)
**核心用途**: 实现 X11 后端核心事件循环，处理 CxOsOp 分发、窗口生命周期、视频播放、重绘调度。

## Types/Structs

### `X11Cx`
X11 上下文。

| 字段 | 类型 | 描述 |
|------|------|------|
| `cx` | `Rc<RefCell<Cx>>` | 共享的 Makepad 核心上下文 |
| `internal_drag_items` | `Option<Arc<Vec<DragItem>>>` | 内部拖拽项 |

## Key Methods/Functions

### `x11_event_loop(cx: Rc<RefCell<Cx>>)`
X11 事件循环入口点，委托给 `X11Cx::event_loop_impl`。

### `X11Cx::event_loop_impl(cx)`
初始化 X11 渲染后端：
1. 设置 `OsType::LinuxWindow`，GPU 性能 Tier1，custom_window_chrome=false
2. 初始化 XlibApp 全局实例
3. 创建 EGL 平台显示（`EGL_PLATFORM_X11_EXT`）
4. 检查 stdin-loop 模式
5. 发送 `Event::Startup`，启动 8ms 定时器，进入事件循环

### `X11Cx::xlib_event_callback(xlib_app, event, opengl_windows) -> EventFlow`
处理 `XlibEvent` 事件：

- **WindowGotFocus**: 重绘所有 pass
- **WindowLostFocus**: 转发失焦事件
- **WindowGeomChange**: 更新几何、DPI
- **WindowClosed**: 关闭子弹出窗口，清理 OpenglWindow
- **PopupDismissed**: 转发弹窗关闭
- **Paint**: 帧事件、绘制事件、着色器编译、重绘
- **MouseDown/MouseUp/Move/Scroll**: DPI 缩放后转发
- **MouseUp (PRIMARY)**: 完成内部拖拽（Drop + DragEnd）
- **WindowDragQuery/CloseRequested**: 转发
- **KeyDown/KeyUp**: 按键处理
- **TextCopy/TextCut**: 剪贴板事件
- **Timer**: 信号处理、控制通道、action、网络、视频播放轮询
- **Drag/Drop/DragEnd**: 拖拽事件转发

### `handle_repaint(opengl_windows)`
计算重绘顺序，对每个窗口/纹理 pass 执行绘制。

### `close_popup_window / close_popup_children`
递归关闭弹出窗口，释放 popup grab。

### `handle_platform_ops(opengl_windows, xlib_app) -> EventFlow`
处理 CxOsOp：

| CxOsOp | 处理方式 |
|--------|---------|
| `CreateWindow` | 创建 OpenglWindow + EGL 表面 |
| `CreatePopupWindow` | 创建弹出窗口，激活 popup grab |
| `CloseWindow` | 关闭窗口，释放 grab |
| `MinimizeWindow` | `XIconifyWindow` |
| `MaximizeWindow` | `XSetWMProperties` 最大化 |
| `RestoreWindow` | 取消最大化 |
| `ResizeWindow` | `set_inner_size` |
| `RepositionWindow` | `set_position` |
| `CopyToClipboard` | `XSetSelectionOwner` |
| `SetPrimarySelection` | `XSetSelectionOwner`（PRIMARY） |
| `SetCursor` | `XDefineCursor` |
| `StartTimer/StopTimer` | `SelectTimers` 操作 |
| `ShowTextIME` | 设置 IME 光标位置，激活 XIC |
| `HideTextIME` | 禁用 XIC |
| `StartDragging` | 设置内部拖拽项 |
| `PrepareVideoPlayback` | V4L2/GStreamer/Software 视频播放（同 Wayland） |
| `PrepareAudioPlayback` | GStreamer 音频播放 |
| 权限操作 | Linux 桌面默认授予所有权限 |
