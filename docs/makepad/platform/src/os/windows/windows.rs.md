# windows.rs — Windows 平台主入口

**文件路径**: platform/src/os/windows/windows.rs (860行)
**核心用途**: Windows 平台的主事件循环和平台操作桥接。处理 Win32 事件到 Makepad 事件的转换、D3D11 窗口管理、视频播放和游戏输入。

## 结构体

### `CxOs`
```rust
pub struct CxOs {
    pub(crate) start_time: Option<Instant>,
    pub(crate) media: CxWindowsMedia,
    pub(crate) d3d11_device: Option<ID3D11Device>,
    pub(crate) game_input_events: GameInputEventChannel,
    pub(crate) windows_game_input: Option<WindowsGameInput>,
    pub(crate) video_players: HashMap<LiveId, WindowsUnifiedVideoPlayer>,
    pub(crate) async_hlsl_compile: AsyncHlslCompile,
}
```

## 关键方法

### `Cx::event_loop` — 主入口
1. 保存 `self_ref`，设置 `os_type = OsType::Windows`
2. 创建 `D3d11Cx`，存储 `ID3D11Device`
3. 如果是 `--stdin-loop` 模式，进入 `stdin_event_loop`
4. 创建 `D3d11Window` 列表
5. 通过 `init_win32_app_global` 注册 `win32_event_callback`
6. 启动 8ms 信号轮询定时器
7. 调用 `Event::Startup`，`redraw_all()`，启动信号轮询
8. 进入 `Win32App::event_loop()`

### `win32_event_callback` — 事件路由

将 `Win32Event` 分发到对应处理：

| Win32Event | 处理逻辑 |
|------------|----------|
| `WindowGotFocus` | 重绘所有窗口的 main pass |
| `WindowResizeLoopStart/Stop` | 在 D3D11Window 上调用 `start_resize/stop_resize` |
| `WindowGeomChange` | 更新窗口几何、DPI 因子（先 `os_dpi_factor` 再 `native_window_geom_to_layout`），重绘 |
| `WindowClosed` | 级联关闭弹出窗口，从 `d3d11_windows` 移除，最后一个窗口触发 `Shutdown` |
| `Paint` | 视频播放帧轮询、定时器事件、重绘、HLSL 着色器编译、`handle_repaint` |
| `MouseDown/Move/Up/Leave` | DPI 缩放校准、手指跟踪、调用事件 |
| `Scroll` | DPI 缩放校准后触发 |
| `WindowDragQuery` | DPI 缩放校准后触发 |
| `Drag/Drop` | 触发并更新 `cycle_drag` |
| `DragEnd` | 发送模拟 `MouseUp`（远离窗口坐标） |
| `KeyDown/Up` | 键盘状态跟踪 |
| `TextCopy/TextCut` | 剪贴板事件 |
| `Timer` | 脚本定时器和事件定时器 |
| `Signal` | 信号处理（MediaFoundation/WASAPI 变更、脚本信号、网络事件、live_edit、控制通道）、递归调用 Paint |

### `handle_repaint`

1. `compute_pass_repaint_order` 计算重绘顺序
2. 每个 pass 根据 parent 类型渲染：
   - `Window`: 在 `D3d11Window` 上 `resize_buffers` + `draw_pass_to_window`
   - `DrawPass`: `draw_pass_to_texture`（离屏渲染）
   - `None`: `draw_pass_to_texture`
   - `Xr`: 无操作

### `handle_platform_ops` — 平台操作派发

处理 `CxOsOp` 队列：

| CxOsOp | 行为 |
|--------|------|
| `CreateWindow` | 创建 D3D11Window，设置初始几何，应用窗口视觉样式 |
| `CreatePopupWindow` | 创建弹出窗口，计算屏幕坐标（相对于父窗口 + DPI），设置 popup 属性 |
| `CloseWindow` | 关闭并移除窗口，最后一个窗口退出 |
| `Minimize/Maximize/Restore` | 窗口状态操作 |
| `ResizeWindow/RepositionWindow` | 窗口位置/大小操作 |
| `SetTopmost` | 置顶设置（支持重新入队，窗口就绪前延迟执行） |
| `SetWindowVisuals` | 应用窗口视觉样式 |
| `CopyToClipboard` | `Win32Window::copy_to_clipboard` |
| `SetCursor` | `win32_app.set_mouse_cursor` |
| `StartTimer/StopTimer` | 定时器管理 |
| `StartDragging` | OLE 拖放操作 |
| `HttpRequest` | 网络请求（`http_start`） |
| `ShowTextIME/HideTextIME` | IME 输入法管理 |
| `CheckPermission/RequestPermission` | 桌面应用默认权限已授予 |
| `PrepareVideoPlayback` | 分配 YUV 纹理，创建 WindowsUnifiedVideoPlayer，发送 `VideoYuvTexturesReady` |
| `Begin/Pause/Resume/Mute/Unmute/Seek/Volume/Rate` | 视频控制 |
| `CleanupVideoPlaybackResources` | 清理播放器 |

### `handle_game_input_events`
- 接收 `GameInputConnectedEvent` 通道事件并分发
- 每帧调用 `WindowsGameInput::poll` 更新设备状态

### CxOsApi 实现

- `init_cx_os`: 记录启动时间，加载 MAKEPAD_PACKAGE_DIR，初始化 `WindowsGameInput`
- `spawn_thread`: `std::thread::spawn`
- `seconds_since_app_start`: `Instant::now() - start_time`

### CxGameInputApi 实现

提供 `game_input_state`、`game_input_states`、`game_input_state_mut`、`game_input_states_mut` 方法访问 WindowsGameInput 的状态数组。

## 视频播放 Paint 事件细节

在 Paint 事件中：
1. 遍历所有 `video_players`
2. `check_prepared` 检查准备状态 → 触发 `VideoPlaybackPrepared` 或 `VideoDecodingError`
3. `poll_frame` 拉取新帧 → 触发 `VideoTextureUpdated`
4. `check_eos` 检查播放结束 → 触发 `VideoPlaybackCompleted`
5. 如有播放中的视频，设置 `new_next_frame` 保持绘制循环活跃
