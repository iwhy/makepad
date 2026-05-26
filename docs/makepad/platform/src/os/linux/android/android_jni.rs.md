# Android JNI 桥接层

## 概述

`android_jni.rs` 是 Makepad Android 平台的 JNI（Java Native Interface）桥接层。它包含：
1. **Java → Rust**：`#[no_mangle] pub extern "C"` 函数，被 Android Java 代码调用
2. **Rust → Java**：`to_java_*` 辅助函数，通过 JNI 调用 Java 方法
3. **线程同步**：Surface 销毁确认机制（`SurfaceAck`）
4. **Choreographer**：Android 帧同步回调管理

## Surface 销毁同步

### `SurfaceAck`（`Arc<(Mutex<bool>, Condvar)>`）
解决 Android 渲染过程中的 SIGSEGV 问题：当 `surfaceDestroyed` 返回后，系统可立即释放缓冲区队列，但渲染线程可能仍在执行 GL 调用。

- `new_surface_ack()` — 创建同步原语
- `signal_surface_ack(ack)` — 渲染线程确认已释放表面
- `wait_surface_ack(ack, timeout)` — JNI 线程等待确认（超时 2 秒，低于 ANR 5 秒阈值）

## Java → Rust 消息

### `FromJavaMessage`
枚举所有从 Java 侧发往 Rust 的消息类型：

| 消息 | 说明 |
|------|------|
| `Init(AndroidParams)` | 初始化参数（缓存/数据路径、密度、模拟器标志、Android 版本等） |
| `SwitchedActivity` | Activity 切换 |
| `BackPressed` | 返回键 |
| `SurfaceCreated/Changed/Destroyed` | Surface 生命周期（`Destroyed` 携带 `SurfaceAck`） |
| `RenderLoop` | 渲染帧（Choreographer 或手动循环触发） |
| `LongClick` | 长按事件 |
| `Touch(Vec<TouchPoint>)` | 触摸事件（含压力、方向、半径） |
| `Character` | 字符输入 |
| `KeyDown/KeyUp` | 按键事件（含 meta 状态） |
| `ResizeTextIME` | IME 键盘显示/隐藏 |
| `SafeAreaInsets` | 安全区域插入 |
| `HttpResponse/HttpRequestError` | HTTP 响应/错误 |
| `WebSocketMessage/Closed/Error` | WebSocket 事件 |
| `MidiDeviceOpened` | MIDI 设备就绪 |
| `PermissionResult` | 权限请求结果 |
| `VideoPlaybackPrepared/Completed/Released/DecodingError` | 视频播放事件 |
| `CameraPreviewSurfaceReady/Destroyed` | 摄像头预览 Surface 事件 |
| `Pause/Resume/Start/Stop/Destroy` | Activity 生命周期 |
| `WindowFocusChanged` | 窗口焦点变化 |
| `ClipboardAction/Paste` | 剪贴板操作 |
| `SelectionHandleDrag` | 选择手柄拖动 |
| `ImeTextStateChanged` | IME 文本状态 |
| `ImeEditorAction` | IME 编辑器动作 |

所有事件通过全局 `MESSAGES_TX`（`mpsc::Sender<FromJavaMessage>`）发送到 Rust 事件循环。

## 全局状态管理

| 函数 | 说明 |
|------|------|
| `send_from_java_message` | 发送消息到事件循环 |
| `from_java_messages_already_set` | 检查通道是否已初始化 |
| `from_java_messages_clear` | 清理通道（关闭时） |
| `jni_set_activity` | 设置 Activity 全局引用 |
| `jni_update_activity` | 更新 Activity 引用 |
| `jni_set_from_java_tx` | 设置消息发送通道 |
| `attach_jni_env` | 附加当前线程到 JVM（获取 JNIEnv） |

## Choreographer 帧同步

### `initChoreographer(device_refresh_rate, sdk_version)`
JNI 入口函数，根据 SDK 版本选择帧同步策略：
- **API 33+**：`AChoreographer_postVsyncCallback`（首选）
- **API 29+**：`AChoreographer_postFrameCallback64`（回退）
- **API < 29 或 `no_android_choreographer`**：手动渲染线程

### `vsync_callback`
Choreographer vsync 回调 → 发送 `RenderLoop` 消息 → 重新注册回调。

### `post_vsync_callback`
通过 `dlsym` 解析的 `Choreographer_postCallbackFn` 注册 vsync 回调。

### `init_simple_render_loop(device_refresh_rate)`
手动帧循环线程：固定频率（如 60Hz/90Hz/120Hz）发送 `RenderLoop` 消息。包含自适应休眠逻辑和关机检测（检查 `from_java_messages_already_set()`）。

## JNI 回调函数（`#[no_mangle]`）

### Activity 生命周期
| 函数 | Java 调用时机 |
|------|--------------|
| `activityOnStart/Resume/Pause/Stop/Destroy` | 对应 Activity 生命周期回调 |
| `activityOnWindowFocusChanged` | 窗口焦点变化 |
| `onAndroidParams` | 初始化参数 |
| `onBackPressed` | 返回键 |

### Surface 操作
| 函数 | 说明 |
|------|------|
| `surfaceOnSurfaceCreated/Changed/Destroyed` | Surface 创建/变化/销毁 |
| `surfaceOnLongClick` | 长按 |
| `surfaceOnTouch` | 触摸事件（解析 `getActionMasked`、`getPointerId`、`getX/Y`、`getOrientation`、`getPressure`、`getTouchMajor/Minor`） |
| `surfaceOnKeyDown/Up/Character` | 按键 / 字符输入 |
| `surfaceOnResizeTextIME` | IME 尺寸变化 |
| `surfaceOnSafeAreaInsets` | 安全区域 |
| `onRenderLoop` | 手动渲染循环 |

### 网络回调
| 函数 | 说明 |
|------|------|
| `onHttpResponse` | HTTP 响应（优先委托 `android_network::try_handle_http_response`） |
| `onHttpRequestError` | HTTP 错误（优先委托 `android_network`） |
| `onWebSocketMessage/Closed/Error` | WebSocket 事件 |

### 视频/摄像头
| 函数 | 说明 |
|------|------|
| `onVideoPlaybackPrepared/Completed/Released/DecodingError` | 视频播放状态 |
| `onH264EncoderPacket/Error` | H.264 编码输出 |
| `onCameraPreviewSurfaceReady/Destroyed` | 摄像头预览 Surface |

### 其他
| 函数 | 说明 |
|------|------|
| `onMidiDeviceOpened` | MIDI 设备就绪 |
| `onPermissionResult/Denied` | 权限结果 |
| `onClipboardAction/Paste` | 剪贴板操作 |
| `onSelectionHandleDrag` | 选择手柄 |
| `onImeTextStateChanged/EditorAction` | IME 状态 |

## Rust → Java 辅助函数

### 窗口 / UI
| 函数 | 说明 |
|------|------|
| `to_java_set_full_screen` | 全屏设置 |
| `to_java_set_system_bar_appearance` | 状态栏外观 |
| `to_java_set_surface_cover_visible` | Surface 覆盖层可见性 |
| `to_java_request_surface_snapshot_refresh` | 刷新表面快照 |
| `to_java_switch_activity` | 切换 Activity |

### 资源
| 函数 | 说明 |
|------|------|
| `to_java_load_asset` | 从 APK 资源加载文件（通过 `AAssetManager`） |

### 键盘 / IME
| 函数 | 说明 |
|------|------|
| `to_java_show_keyboard` | 显示/隐藏键盘 |
| `to_java_configure_keyboard` | 配置键盘模式、自动大写、自动纠正、返回键类型 |
| `to_java_update_ime_text_state` | 更新 IME 文本状态 |

### 剪贴板
| 函数 | 说明 |
|------|------|
| `to_java_copy_to_clipboard` | 复制 |
| `to_java_paste_from_clipboard` | 粘贴 |
| `to_java_show_clipboard_actions` | 显示剪贴板操作菜单 |
| `to_java_dismiss_clipboard_actions` | 关闭菜单 |
| `to_java_show_selection_handles` | 显示选择手柄 |
| `to_java_update_selection_handles` | 更新手柄 |
| `to_java_hide_selection_handles` | 隐藏手柄 |

### 网络
| 函数 | 说明 |
|------|------|
| `to_java_http_request` | HTTP 请求 |
| `to_java_websocket_open/send_message/close` | WebSocket 操作 |
| `to_java_socket_stream_open/read/write/close/set_read_timeout/set_write_timeout` | TCP Socket 操作 |

### 音频 / MIDI
| 函数 | 说明 |
|------|------|
| `to_java_get_audio_devices` | 枚举音频设备 |
| `to_java_open_all_midi_devices` | 打开所有 MIDI 设备 |

### 摄像头
| 函数 | 说明 |
|------|------|
| `to_java_attach_camera_preview` | 附加摄像头原生预览 |
| `to_java_update_camera_preview` | 更新预览区域 |
| `to_java_detach_camera_preview` | 分离预览 |

### 视频播放
| 函数 | 说明 |
|------|------|
| `to_java_prepare_video_playback` | 准备播放（支持内存/网络/文件源） |
| `to_java_update_tex_image` | 更新外部纹理 |
| `to_java_begin/pause/resume/mute/unmute/seek_video_playback` | 播放控制 |
| `to_java_get_video_position` | 获取播放位置 |
| `to_java_cleanup_video_playback_resources` | 清理资源 |
| `to_java_cleanup_video_decoder_ref` | 释放解码器引用 |

### 权限
| 函数 | 说明 |
|------|------|
| `to_java_check_permission` | 检查权限状态 |
| `to_java_request_permission` | 请求权限 |

### 编解码
| 函数 | 说明 |
|------|------|
| `to_java_query_h264_codec_support` | 查询 H.264 硬件/软件编码支持（返回 `AndroidH264CodecProbe`） |

## 内部辅助函数

| 函数 | 说明 |
|------|------|
| `jstring_to_string` | Java String → Rust String |
| `java_string_array_to_vec` | Java String[] → Vec<String> |
| `java_byte_array_to_vec` | Java byte[] → Vec<u8> |
| `fetch_activity_handle` | 创建 Activity 全局引用 |
| `create_native_window` | Java Surface → ANativeWindow |
| `get_intent_string_extra` | 获取 Intent Extra |
| `get_persisted_string_pref` | 读取 SharedPreferences |
| `persist_string_pref` | 写入 SharedPreferences |

## Android Studio 环境变量

`apply_studio_env_from_activity()` 从 Intent Extra 或 SharedPreferences 读取 `STUDIO_HOST` 和 `STUDIO_CRATE` 环境变量，支持 Makepad Studio 远程开发。

## 实现说明

- `MESSAGES_TX` 使用 `Mutex<Option<Sender>>`，在清理时置 `None` 以阻止后续消息发送。
- 所有网络回调优先委托给 `android_network` 模块的 shim backend，委托失败后才 fallback 到 `FromJavaMessage` 通道。
- 权限状态码：0=NotDetermined, 1=Granted, 2=DeniedCanRetry, 3=DeniedPermanent。
- `call_method!` 宏在 `ndk_utils.rs` 中定义，通过 `static AtomicPtr` 缓存 JNI MethodID。
