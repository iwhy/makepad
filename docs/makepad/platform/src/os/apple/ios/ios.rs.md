# ios.rs — iOS 平台核心实现

**文件路径**: `platform/src/os/apple/ios/ios.rs` (1682 行)
**核心作用**: Cx 的事件循环和平台操作处理、摄像头播放器（软件解码和原生预览）、视频播放集成、权限管理、`CxOs` 状态结构体定义。

## 摄像头播放器

### IosCameraPlayer（软件解码路径）

通过 `AVCapture` 获取摄像头帧，使用 `AppleYuvMetal` 上传到 Metal 纹理。

| 字段 | 说明 |
|------|------|
| `video_id` | 视频标识 |
| `tex_y_id/u_id/v_id` | YUV 三平面纹理 ID |
| `latest_nv12` | 最新 NV12 格式帧（由像素缓冲回调填充） |
| `i420_frames` | I420 帧环形缓冲区（由帧回调填充） |
| `yuv_metal` | Metal YUV 转换器 |
| `camera_access` | `AvCaptureAccess` 引用 |

**初始化流程** (`new`):
1. 注册 I420 帧回调（`CameraFrameInputFn`）和 NV12 像素缓冲回调（`CameraPixelBufferInputFn`）
2. I420 帧发布到环形缓冲区，NV12 帧保持最新引用
3. 创建 `AppleYuvMetal` 实例

**关键方法**:
- `check_prepared()` — 检测第一帧到达，构造 `PlaybackPrepared` 事件（宽高、音视频轨道信息）
- `poll_frame(textures)` — 拉取最新的 NV12 或 I420 帧，调用 `yuv_metal.wrap_nv12_cv_pixel_buffer`（NV12）或 `yuv_metal.upload_r8_plane`（I420）更新纹理，返回是否有新帧
- `yuv_biplanar()` — 当前帧是否为双平面格式（NV12 vs I420）
- `cleanup()` — 释放帧缓冲、清理 YUV Metal、取消摄像头订阅

### IosNativeCameraPreview（原生预览路径）

使用 `AVCaptureVideoPreviewLayer` 直接在 UIKit 视图层级中显示摄像头画面。

| 字段 | 说明 |
|------|------|
| `video_id` | 视频标识 |
| `input_id/format_id` | 摄像头输入/格式 |
| `camera_access` | `AvCaptureAccess` 引用 |

**关键方法**:
- `check_prepared()` — 构造 `PlaybackPrepared` 事件（从 `camera_access.format_size` 获取宽高）
- `session()` — 获取 `AVCaptureSession` 用于创建原生预览层
- `cleanup()` — 取消摄像头订阅

### 辅助函数

- `register_ios_camera_subscription` — 注册摄像头帧订阅
- `unregister_ios_camera_subscription` — 取消摄像头帧订阅

## Cx 平台实现

### 事件循环 `Cx::event_loop(cx)`

iOS 应用启动入口：

1. 获取设备信息（型号、系统版本）和 data path，设置 `OsType::Ios`
2. 创建 Metal 上下文
3. 初始化 Apple 全局类和 iOS 应用全局状态
4. 设置事件回调（`ios_event_callback`）
5. 调用 `IosApp::event_loop()` 启动 `UIApplicationMain`

### 事件回调 `ios_event_callback(event, metal_cx)`

将 `IosEvent` 分派到 Makepad 标准事件系统：

| IosEvent | 处理逻辑 |
|----------|----------|
| `Init` | 启动 timer (8ms)、启动 Studio WebSocket、设置 safe area insets、触发 `Startup`+`Foreground` |
| `Foreground/Background` | 触发对应事件 |
| `Pause/Resume` | 触发对应事件 |
| `Shutdown` | 触发后返回 `EventFlow::Exit` |
| `WindowGeomChange` | 更新 dpi、转换坐标为 layout 空间 |
| `Paint` | 轮询视频/摄像头播放器、处理 next frame、触发 draw 和 repaint |
| `TouchUpdate` | 转换坐标、检测弹出窗口外点击关闭、处理内部拖放、发送 `TouchUpdate`/`Drag`/`Drop`/`DragEnd` |
| `LongPress` | 转换坐标，发送 `LongPress` 事件 |
| `MouseDown/Move/Up` | 转换坐标、弹出窗口关闭检测、tap count、hover 区域循环 |
| `Scroll` | 转换坐标 |
| `TextInput/RangeReplace` | 直接透传 |
| `KeyDown/Up` | 通过 `keyboard.process_key_down/up` 处理 |
| `TextCopy/Cut` | 透传 |
| `Timer(id=0)` | 处理虚拟键盘事件、批量处理排队文本事件、检测信号、运行 live edit、处理网络/权限事件 |
| `Timer(id!=0)` | 脚本定时器处理 |

**关键处理** - timer id=0（主 tick）:
1. 检查并发送排队虚拟键盘事件（含 IME 关闭保护）
2. 批量处理 `queued_text_events`（文本输入、范围替换、选择变化、按键事件）
3. 检测 UI/Action 信号
4. 运行 live edit、网络事件、权限事件

**脏帧检测** — 事件处理后检测是否需要继续轮询：
- passes dirty / need redrawing / next frames 非空
- paint_dirty / demo_time_repaint 标志
- 视频/摄像头播放器非空

### 平台操作处理 `handle_platform_ops(metal_cx)`

处理 `CxOsOp` 枚举的 iOS 实现：

| 操作 | 实现 |
|------|------|
| `CreateWindow` | 从 IosApp 获取窗口几何 |
| `CreatePopupWindow` | 设置弹出窗口几何和属性 |
| `ShowTextIME/HideTextIME` | 设置 IME 位置/键盘配置，显示/隐藏键盘 |
| `SyncImeState` | 同步文本和选择到 IME |
| `StartTimer/StopTimer` | 管理 NSTimer |
| `ShowClipboardActions/HideClipboardActions` | 调用 IosApp 的剪贴板菜单 |
| `CopyToClipboard` | 写入 UIPasteboard |
| `ShowSelectionHandles/Update/Hide` | 管理选择手柄 |
| `FullscreenWindow/NormalizeWindow` | 设置/取消全屏 |
| `AttachCameraNativePreview/Update/Detach` | 管理 AVCaptureVideoPreviewLayer |
| `SpawnSystemBrowser/Update/Detach/Close` | 管理 IosSystemBrowser |
| `PrepareVideoPlayback` | 根据 `CameraPreviewMode` 创建 `IosCameraPlayer`（软件）或 `IosNativeCameraPreview`（原生）或 `AppleUnifiedVideoPlayer`（视频文件/流），分配 YUV 纹理并触发 `VideoYuvTexturesReady` |
| `Begin/Pause/Resume/Mute/Unmute/Seek/Volume/Rate` | 控制视频播放器 |
| `CleanupVideoPlaybackResources` | 清理播放器并触发 `VideoPlaybackResourcesReleased` |
| `PrepareAudioPlayback` | 创建无纹理的 AppleUnifiedVideoPlayer |
| `CloseWindow` | 重置弹出窗口状态 |
| `StartDragging` | 设置内部拖拽项目 |
| `CheckPermission/RequestPermission` | 检查/请求音视频权限 |

### 权限管理

- `check_audio_permission_status()` — 通过 `AVAudioSession.recordPermission` 检查
- `check_camera_permission_status()` — 通过 `AVCaptureDevice.authorizationStatusForMediaType` 检查
- `ios_request_audio_permission()` — 调用 `AVAudioSession.requestRecordPermission`（ObjC block 回调通过 channel 发送结果）
- `ios_request_camera_permission()` — 调用 `AVCaptureDevice.requestAccessForMediaType`（ObjC block 回调通过 channel 发送结果）

### 渲染 `handle_repaint(metal_cx)`

按顺序绘制所有需要重绘的 draw pass：
1. 计算 pass 重绘顺序
2. 更新 pass 时间
3. 对 Window pass：使用 `MTKView` 模式绘制，弹出窗口作为叠加层绘制在同一 `MTKView` 上
4. 对子 pass 和 None pass：使用 `Texture` 模式绘制
5. 如果摄像头已初始化，逐 index 调用 `video_encoder_capture_texture_frame`

### CxOsApi 实现

| 方法 | 实现 |
|------|------|
| `init_cx_os` | 设置 start_time、package root（生产环境 `"makepad"`、模拟器加载依赖） |
| `spawn_thread` | `std::thread::spawn` |
| `seconds_since_app_start` | 从 start_time 计算 |
| `open_url` | 错误日志（未实现） |
| `max_texture_width` | 16384 |

### 弹出窗口管理

- `find_popup_to_dismiss_on_touch(touches)` — 检查触摸开始点是否在所有弹出窗口外
- `find_popup_to_dismiss_on_mouse(abs)` — 鼠标版
- `dismiss_popup_window(window_id, reason)` — 递归关闭子弹出窗口，触发 `PopupDismissed` 和 `WindowClosed` 事件

## CxOs 结构体

```rust
pub struct CxOs {
    pub start_time: Option<Instant>,
    pub media: CxAppleMedia,
    pub bytes_written: usize,
    pub draw_calls_done: usize,
    // ... 统计字段
    pub permission_response: PermissionResultChannel,
    pub apple_game_input: Option<AppleGameInput>,
    pub video_players: HashMap<LiveId, AppleUnifiedVideoPlayer>,
    pub camera_players: HashMap<LiveId, IosCameraPlayer>,
    pub native_camera_previews: HashMap<LiveId, IosNativeCameraPreview>,
    pub system_browsers: HashMap<LiveId, IosSystemBrowser>,
    pub internal_drag_items: Option<Arc<Vec<DragItem>>>,
}
```

### PermissionResultChannel

```rust
pub struct PermissionResultChannel {
    pub receiver: Receiver<PermissionResult>,
    pub sender: Sender<PermissionResult>,
}
```

使用 `std::sync::mpsc::channel` 实现线程安全的权限结果传递（从 ObjC block 回调到 Rust 事件系统）。
