# Android 平台主实现

## 概述

`android.rs` 是 Makepad Android 平台的核心实现文件（3455 行），连接 Java 侧 Activity/Surface/输入事件 与 Rust 渲染/事件循环。它包含主事件循环、消息处理、OpenGL/Vulkan 渲染、平台操作分发和 XR 支持。

## 全局常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `ANDROID_XR_BUFFER_SCALE_MIN` | 0.75 | XR 渲染缓冲最小缩放 |
| `ANDROID_XR_BUFFER_SCALE_DEFAULT` | 1.4 | XR 渲染缓冲默认缩放 |
| `ANDROID_XR_BUFFER_SCALE_MAX` | 1.5 | XR 渲染缓冲最大缩放 |
| `ANDROID_XR_MULTISAMPLES` | 4 | XR 多重采样数 |
| `ANDROID_XR_FIXED_FOVEATION_LEVEL` | 3 | XR 固定注视点渲染级别 |

## 全局辅助函数

### `android_debug_log(prio, msg)`
通过 `__android_log_write` 将日志写入 Android logcat（tag="Makepad"）。

### `android_panic_summary(info) -> String`
格式化 panic 信息，包含线程名、位置、payload 和完整 backtrace。

### `install_android_panic_hook()`
设置全局 panic hook：panic 时写入 logcat 再调用之前的 hook。

### `set_current_thread_priority(priority)`
通过 `setpriority(PRIO_PROCESS)` 设置当前线程优先级：
- Normal → nice 0 | Utility → 5 | Background → 10 | Idle → 15

### `string_to_permission(str) -> Option<Permission>`
将 Android 权限字符串转换为 Rust `Permission` 枚举：
- `"android.permission.RECORD_AUDIO"` → `AudioInput`
- `"android.permission.CAMERA"` → `Camera`
- `"horizonos.permission.HEADSET_CAMERA"` → `HeadsetCamera`
- `"com.oculus.permission.USE_SCENE"` → `SceneAccess`

### `to_android_permission(permission) -> &str`
反向映射。

## 核心类型

### `CxOs`
Android 平台状态结构体，包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `first_after_resize` | `bool` | 调整大小后首次渲染标志 |
| `needs_first_draw` | `bool` | 表面可用后需要首次全重绘 |
| `hide_surface_cover_after_first_present` | `bool` | 首次呈现后隐藏 Java 覆盖层 |
| `refresh_surface_snapshot_after_first_present` | `bool` | 首次呈现后刷新任务快照 |
| `display_size` | `Vec2d` | 显示尺寸（物理像素） |
| `dpi_factor` | `f64` | DPI 密度因子 |
| `native_safe_area_insets` | `SafeAreaInsets` | 安全区域（系统栏/刘海/打孔屏） |
| `last_ime_height` | `f64` | 上一次 IME 高度（去重用） |
| `last_ime_visible` | `bool` | IME 可见状态（去重用） |
| `last_ime_config` | `Option<TextInputConfig>` | 上次发给 IME 的配置（去重用） |
| `frame_time` | `i64` | 帧时间 |
| `quit` | `bool` | 退出标志 |
| `start_time` | `Instant` | 应用启动时间 |
| `timers` | `PollTimers` | 定时器管理器 |
| `display` | `Option<CxAndroidDisplay>` | EGL/Vulkan 显示状态 |
| `surface_alive` | `bool` | 渲染表面是否有效 |
| `vulkan` | `Option<CxVulkan>` | Vulkan 后端（仅 `use_vulkan`） |
| `media` | `CxAndroidMedia` | 媒体子系统 |
| `video_surfaces` | `HashMap<LiveId, jobject>` | SurfaceTexture 引用 |
| `video_configs` | `HashMap<LiveId, AndroidVideoConfig>` | 视频配置 |
| `camera_players` | `HashMap<LiveId, AndroidCameraPlayer>` | Android NDK 摄像头播放器 |
| `pending_camera_preview_windows` | `HashMap<LiveId, *mut ANativeWindow>` | 待处理的预览窗口 |
| `software_video_players` | `HashMap<LiveId, AndroidSoftwarePlayer>` | 软件视频播放器（rav1d） |
| `websocket_parsers` | `HashMap<u64, WebSocketImpl>` | WebSocket 解析器 |
| `internal_drag_items` | `Option<Arc<Vec<DragItem>>>` | 内部拖拽项 |
| `openxr` | `CxOpenXr` | OpenXR 状态 |
| `activity_thread_id` | `Option<u64>` | Activity 线程 ID |
| `render_thread_id` | `Option<u64>` | 渲染线程 ID |
| `ignore_destroy` | `bool` | 忽略 Destroy 事件（XR 模式） |
| `in_xr_mode` | `bool` | XR 模式标志 |
| `xr_buffer_scale_*` | `f32` | XR 缓冲缩放（请求/激活） |
| `xr_display_refresh_rate_active_hz` | `Option<f32>` | XR 显示刷新率 |
| `xr_effective_*` | `Option<f64>` | XR 有效帧率/时间 |
| `xr_frame_cpu_time_ms` | `Option<f64>` | XR CPU 帧时间 |
| `xr_render_cpu_time_ms` | `Option<f64>` | XR 渲染 CPU 时间 |
| `xr_depth_readback_cpu_time_ms` | `Option<f64>` | XR 深度回读 CPU 时间 |
| `xr_frame_cpu_breakdown` | `Option<XrFrameCpuBreakdown>` | XR CPU 细分 |
| `xr_pending_surface_window/width/height` | - | XR 待处理表面（仅 `use_vulkan`） |
| `xr_retry_surface_after_destroy` | `bool` | 销毁后重试 XR 会话 |

### `CxAndroidDisplay`
EGL 显示状态：

| 字段 | 说明 |
|------|------|
| `libegl` | EGL 库句柄 |
| `libgl` | OpenGL 库句柄 |
| `egl_display` | EGL 显示连接 |
| `egl_config` | EGL 配置 |
| `egl_context` | EGL 渲染上下文 |
| `surface` | EGL 表面 |
| `window` | `ANativeWindow` 指针 |

方法：
- `is_surface_alive()` — 表面是否非空
- `make_current()` — 绑定 EGL 上下文（panic 版）
- `try_make_current()` — 绑定 EGL 上下文（fallible 版）
- `destroy_surface()` — 销毁 EGL 表面（仅 OpenGL）
- `update_surface(window)` — 更新 EGL 表面（仅 OpenGL）

### `AndroidSoftwarePlayer`
软件视频播放器状态：
- `player` — `PlaybackSessionHandle`
- `tex_y_id` / `tex_u_id` / `tex_v_id` — YUV 纹理 ID
- `yuv_matrix` — YUV 矩阵

## 主事件循环

### `Cx::main_loop(from_java_rx)`
Android 平台主入口（第 285-420 行）：

1. 设置 `OsType::Android` 和 `GpuPerformance::Tier1`
2. 初始化 `display_context`（屏幕尺寸、安全区域）
3. 调用 `Event::Startup` + `redraw_all()`
4. 启动网络/文件观察器
5. 进入主消息循环：

#### RenderLoop 消息处理流程
1. 阻塞等待 `from_java_rx.recv()`
2. 收到 `RenderLoop` 后清空待处理队列：
   - **触摸事件合并**：连续的纯 Move 触摸事件会被合并（只保留最新的），非 Move 事件会打断合并并清空延迟的 Move
3. 处理其他事件（`handle_other_events`）
4. XR 模式检查：等待会话就绪
5. 检查 `pending_script_reapply` / `pending_live_edit_request`
6. 安全检查：**`has_drawable_surface()`** 确保表面有效才渲染
7. 首次绘制标记处理
8. `handle_drawing()` 执行渲染

#### 非 RenderLoop 消息
- 其他消息直接交给 `handle_message` 处理
- 立即执行 `handle_platform_ops()`（确保 IME 状态同步）

#### 退出清理
- 销毁 XR 实例（Vulkan/OpenGL 分支）
- 调用 `from_java_messages_clear()`

### `Cx::android_entry(activity, startup)`
静态方法，应用启动入口：
1. 获取 Activity 线程 ID 和句柄
2. 检查是否已有运行中的实例（Activity 切换场景）
3. 创建 `mpsc::channel`
4. 设置 panic hook
5. 设置 JNI 状态
6. 启动渲染线程：
   - 附加当前线程到 JVM
   - 加载 LibEGL + LibGL
   - 创建 EGL 上下文 + 表面（Vulkan 模式下创建 pbuffer 表面）
   - 创建 Vulkan 后端（可选）
   - 调用 `main_loop(from_java_rx)`
   - 清理资源（`eglDestroySurface/Context/Terminate`）

## 消息处理

### `handle_message(msg)`
处理所有 `FromJavaMessage` 变体（第 475-1356 行）：

| 消息 | 处理 |
|------|------|
| `SwitchedActivity` | 更新 Activity 引用，初始化 XR |
| `BackPressed` | 派发 `BackPressed` 事件 |
| `SurfaceCreated` | 更新 EGL/Vulkan 表面，标记重绘 |
| `SurfaceDestroyed` | 清除 `surface_alive` → 销毁 EGL 表面 → 释放窗口 → 信号确认 → XR 回话重试 |
| `SurfaceChanged` | 更新表面 + 窗口几何 + DPI + 派发 `WindowGeomChange` + 重绘 |
| `LongClick` | 转换坐标 → `LongPressEvent` |
| `Touch` | 坐标转换 → 弹出窗口关闭检查 → `TouchUpdateEvent` + 内部拖拽 |
| `Character` | `TextInput` 事件 |
| `KeyDown` / `KeyUp` | 键码映射 + Ctrl+C/X/V 剪贴板快捷键 + Back 键处理 |
| `ResizeTextIME` | IME 高度跟踪 + 去重 + `VirtualKeyboardEvent` |
| `HttpResponse` / `HttpRequestError` | 网络响应分发 |
| `WebSocket*` | WebSocket 消息解析和分发 |
| `MidiDeviceOpened` | 更新 MIDI 设备状态 |
| `PermissionResult` | 权限状态转换和分发 |
| `VideoPlaybackPrepared/Completed/Released/DecodingError` | 视频播放事件 |
| `CameraPreviewSurfaceReady/Destroyed` | 摄像头预览窗口管理 |
| `Pause/Resume/Start/Stop/Destroy` | Activity 生命周期事件 |
| `WindowFocusChanged` | 焦点事件 |
| `ClipboardAction` | 剪贴板复制/剪切/全选 |
| `ClipboardPaste` | 粘贴文本输入 |
| `SelectionHandleDrag` | 选择手柄拖动 |
| `ImeTextStateChanged` | UTF-16 ↔ UTF-8 偏移转换 → 全文本状态同步 |
| `ImeEditorAction` | 编辑器动作事件 |
| `SafeAreaInsets` | 安全区域更新 + WindowGeomChange 检查 |
| `Init` | 忽略（已在入口处处理） |

### `handle_other_events()`
每帧执行的辅助事件处理（第 1504-1556 行）：
1. 定时器分发
2. 信号处理（UI 信号、Action 信号）
3. 网络事件分发
4. Studio 运行视图帧结果刷新
5. 视频 SurfaceTexture 更新轮询
6. 摄像头播放器轮询（`poll_camera_players`）
7. 软件视频播放器轮询（`poll_software_video_players`）
8. Live edit 处理
9. 平台操作分发

## 渲染

### `handle_drawing()`
检查是否需要进行渲染（第 1358-1380 行）：
- 脏通道、需要重绘、next_frames、demo 重绘
- 调用 `compile_shaders_for_active_backend`

### `compile_shaders_for_active_backend()`
- **Vulkan**：跳过（SPIR-V 编译在 draw-shader 创建时完成）
- **OpenGL**：调用 `opengl_compile_shaders()`

### `draw_pass_to_window_for_active_backend(draw_pass_id)`
渲染到窗口表面（第 1396-1431 行）：
- 安全检查 `has_drawable_surface()`
- **Vulkan**：`vulkan.draw_pass_and_present()` → 隐藏覆盖层 / 刷新快照
- **OpenGL**：`draw_pass_to_fullscreen()`

### `draw_pass_to_texture_for_active_backend(draw_pass_id)`
渲染到纹理（第 1433-1454 行）：
- 安全检查 `has_drawable_surface()`
- **Vulkan**：`vulkan.draw_pass_to_texture()`
- **OpenGL**：`draw_pass_to_texture()`

### `present_window_for_active_backend()`
呈现窗口（第 1456-1500 行）：
- `eglSwapBuffers` + 隐藏覆盖层/刷新快照

### `draw_pass_to_fullscreen(draw_pass_id)`
全屏 OpenGL 渲染（第 2081-2160 行）：
1. 安全检查 `has_drawable_surface()`
2. `try_make_current()`（每帧重新绑定上下文防漂移）
3. `glViewport` + `glClear`（颜色/深度）+ 默认混合模式
4. `render_view` 渲染 UI
5. zbias 步进

### `handle_repaint()`
重绘管理器（第 2162-2235 行）：
1. 计算渲染通道顺序
2. 对每个 Window 通道：渲染窗口 → 渲染弹出层覆盖 → GPU 指标采集 → 呈现
3. 对每个纹理通道：`draw_pass_to_texture`
4. 视频编码器纹理帧捕获

## 平台操作处理

### `handle_platform_ops() -> EventFlow`
处理平台操作队列（第 2237-2895 行）：

| 操作 | 处理 |
|------|------|
| `CreateWindow` | 设置窗口几何（全屏 + DPI + 安全区域） |
| `CreatePopupWindow` | 创建弹出窗口（位置/尺寸/键盘捕获） |
| `CloseWindow` | 关闭弹出窗口 |
| `StartTimer` / `StopTimer` | 定时器管理 |
| `ShowTextIME` | IME 配置去重 + 显示键盘 |
| `HideTextIME` | 隐藏键盘 + 清除配置 |
| `SyncImeState` | UTF-16 偏移转换 → `update_ime_text_state` |
| `CopyToClipboard` | `to_java_copy_to_clipboard` |
| `ShowSelectionHandles` | 坐标转换 → `to_java_show_selection_handles` |
| `UpdateSelectionHandles` | `to_java_update_selection_handles` |
| `HideSelectionHandles` | `to_java_hide_selection_handles` |
| `ShowClipboardActions` | `to_java_show_clipboard_actions` |
| `HideClipboardActions` | `to_java_dismiss_clipboard_actions` |

### 摄像头相关
| 操作 | 处理 |
|------|------|
| `AttachCameraNativePreview` | 布局坐标 → 物理像素 → `to_java_attach_camera_preview` |
| `UpdateCameraNativePreview` | `to_java_update_camera_preview` |
| `DetachCameraNativePreview` | `to_java_detach_camera_preview` + 释放窗口 |

### 权限
| 操作 | 处理 |
|------|------|
| `CheckPermission` | JNI 检查 → `PermissionResult` |
| `RequestPermission` | 已授权直接返回，否则 `to_java_request_permission` |

### 视频播放
| 操作 | 处理 |
|------|------|
| `PrepareVideoPlayback` | 摄像头源 → `AndroidCameraPlayer`；文件源 → 软件解码器或 JNI 原生播放 |
| `Begin/Pause/Resume/Mute/Unmute/Seek/Cleanup/Volume/Rate` | 委托给 `AndroidCameraPlayer` 或 `PlaybackSessionHandle` 或 JNI |
| `PrepareAudioPlayback` | 当前仅占位（TODO） |

### XR 操作
| 操作 | 处理 |
|------|------|
| `XrStartPresenting` | 设置 XR 参数、切换 Activity、激活 XR 模式 |
| `XrStopPresenting` | 清理 XR 状态、切换回主 Activity |
| `XrSetRenderScale` | 设置缓冲缩放 |
| `XrAdvertiseAnchor` | 向 OpenXR 注册锚点 |
| `XrSetLocalAnchor` | 设置本地锚点 |
| `XrSetLocalFloor` | 设置本地地面高度 |
| `XrDiscoverAnchor` | 发现锚点 |

### 窗口操作
| 操作 | 处理 |
|------|------|
| `FullscreenWindow` / `NormalizeWindow` | `to_java_set_full_screen` |
| `SetSystemBarDarkIcons` | `to_java_set_system_bar_appearance` |
| `StartDragging` | 保存 `internal_drag_items` |

### 未实现（记录错误）
`SetPrimarySelection`、`AccessibilityUpdate`、`SetCursor`、`open_url`、`PrepareAudioPlayback` 等。

## CxOsApi 实现

| 方法 | 实现 |
|------|------|
| `init_cx_os` | 安装网络后端 shim + 设置 `package_root` |
| `spawn_thread` | `std::thread::spawn` |
| `seconds_since_app_start` | `Instant::now() - start_time` |
| `open_url` | 未实现（记录错误） |
| `in_xr_mode` | 返回 `self.os.in_xr_mode` |
| `micro_zbias_step` | XR 模式 `-0.0001`，普通模式 `0.00001` |
| `xr_render_scale` | XR 模式返回 `buffer_scale_active` |
| `xr_gpu_frame_time_ms` | Vulkan 模式委托给 `vulkan.last_openxr_gpu_frame_time_ms` |
| `xr_*` 系列 | 返回 `CxOs` 中存储的各 XR 指标 |

## XR 支持（OpenXR）

### `current_android_xr_options() -> CxOpenXrOptions`
返回当前 XR 选项（缓冲缩放、多重采样、注视点级别）。

### `replace_xr_pending_surface` / `clear_xr_pending_surface`
管理 XR 待处理表面状态（仅 `use_vulkan`）。

### `try_create_xr_session_for_surface(window, width, height, reason)`
尝试为 XR 创建会话（第 212-279 行）：
1. 检查是否已在 XR 模式且无会话
2. 首次创建 OpenXR 实例
3. 备份/暂停 Vulkan 表面
4. 创建 Vulkan 后端
5. 使用当前选项创建 OpenXR 会话

### `openxr_render_loop(from_java_rx) -> bool`
XR 渲染循环（在主循环中调用），处理 Orientation/Input/Space/Passthrough/Frame 和 UI 渲染。

## `CxOs::has_drawable_surface()`
安全检查方法（第 3319-3353 行）：
- **Vulkan + XR 活动**：直接检查 `vulkan.is_some()`（独立于 Android 表面）
- **OpenGL**：`surface_alive` + `display.is_surface_alive()`
- **Vulkan 非 XR**：`surface_alive` + `vulkan.has_drawable_surface()`

## 弹出窗口管理

### `find_popup_to_dismiss_on_touch(touches) -> Option<WindowId>`
在触摸开始位置检查是否有弹出窗口需要关闭。

### `dismiss_popup_window(window_id, reason)`
递归关闭弹出窗口及其子窗口。

## 权限处理

### `check_android_permission_status(permission) -> PermissionStatus`
通过 JNI 检查权限状态，码值映射：
- 0 → `NotDetermined`
- 1 → `Granted`
- 2 → `DeniedCanRetry`
- 其他 → `DeniedPermanent`

### `handle_permission_check/request`
检查/请求权限，已授权直接返回，否则通过 JNI 弹出系统权限对话框。

## 视频 / 摄像头轮询

### `get_video_updates() -> Vec<LiveId>`
轮询 SurfaceTexture 更新，每帧调用 `to_java_update_tex_image`。

### `poll_camera_players()`
轮询 `AndroidCameraPlayer`：
1. 检查准备状态
2. Vulkan + AHardwareBuffer → `update_video_external_hardware_buffer_texture`
3. OpenGL → `player.poll_frame` → 上传到纹理
4. Vulkan 导入失败 → fallback 到 CPU YUV

### `poll_software_video_players()`
轮询软件视频播放器：
1. 检查准备状态
2. `poll_frame` → `take_yuv_frame` → `upload_yuv_to_gl`
3. 检查 EOS

## 依赖加载

### `android_load_dependencies()`
通过 `to_java_load_asset` 从 APK assets 加载依赖文件。

## 实现说明

- 事件循环使用 `mpsc::Receiver::recv()` 阻塞等待 Java 消息，由 Choreographer vsync 驱动 `RenderLoop`。
- 触摸事件合并策略：纯 Move 事件延迟到帧边界处理，避免冗余事件。
- Surface 生命周期管理遵循严格的顺序：`surface_alive = false` → EGL 销毁 → `signal_surface_ack`。
- `try_make_current()` 每帧调用用于防止 EGL 上下文漂移（Mali/Adreno 驱动安全）。
- 视频解码有三条路径：原生 MediaCodec → 软件解码器（rav1d）→ 错误事件。
- 摄像头支持 NDK Camera2 API（`android_camera.rs`）和摄像预览 Surface（`android_camera_player.rs`）两种模式。
