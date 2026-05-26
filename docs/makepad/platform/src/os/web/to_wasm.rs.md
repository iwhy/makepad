# to_wasm.rs — JS→Rust (ToWasm) 消息类型定义

**文件路径**: `/home/ubuntu/_github/makepad/platform/src/os/web/to_wasm.rs` (641 行)
**核心作用**: 定义所有从 JavaScript 发送到 Rust 的消息结构体，涵盖窗口事件、输入事件、网络响应、媒体事件等，以及相应的类型转换实现。

## 消息结构体分类

### 初始化与窗口

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `WGpuInfo` | `min_uniform_vectors`, `vendor`, `renderer` | GPU 信息描述 |
| `WBrowserInfo` | `protocol`, `hostname`, `host`, `pathname`, `search`, `hash`, `has_thread_support`, `small_font_aliases` | 浏览器环境信息 |
| `WWindowInfo` | `is_fullscreen`, `can_fullscreen`, `xr_is_presenting`, `vr_supported`, `ar_supported`, `dpi_factor`, `inner_width`, `inner_height` | 窗口几何信息 |
| `WXrCapabilities` | `vr_supported`, `ar_supported` | XR 能力描述 |
| `ToWasmInit` | `gpu_info`, `cpu_cores`, `xr_capabilities`, `browser_info`, `window_info` | 应用初始化消息 |
| `ToWasmResizeWindow` | `window_info` | 窗口大小变化 |

### 类型转换实现

- `WBrowserInfo → OsType::Web(WebParams{...})` — 浏览器信息转操作系统类型
- `WWindowInfo → WindowGeom` — 窗口信息转内部几何数据结构
- `WXrCapabilities → XrCapabilities` — XR 能力转换

### 帧与定时器

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `ToWasmAnimationFrame` | `time: f64` | 动画帧回调控时 |
| `ToWasmTimerFired` | `timer_id: usize` | 定时器触发 |
| `ToWasmSignal` | `flags: u32` | 信号通知（bit 0=UI 信号, bit 1=Action 信号） |
| `ToWasmAppLifecycle` | `state: u32` | 应用生命周期状态（0=Foreground,1=Background,2=Pause,3=Resume,4=Shutdown） |
| `ToWasmPaintDirty` | — | 标记绘制脏 |
| `ToWasmRedrawAll` | — | 请求全局重绘 |
| `ToWasmLiveFileChange` | `file_name`, `content` | 热重载文件变更 |
| `ToWasmLocationChange` | `pathname`, `search`, `hash` | 浏览器 URL 变化 |

### 触摸事件

| 结构体/类型 | 字段 | 用途 |
|-------------|------|------|
| `WTouchPoint` | `time`, `state`(0=stable,1=start,2=move,3=end), `x`, `y`, `radius_x/y`, `rotation_angle`, `force`, `uid` | 触摸点数据 |
| `ToWasmTouchUpdate` | `modifiers`, `time`, `touches: Vec<WTouchPoint>` | 触摸事件更新 |

转换：`WTouchPoint → TouchPoint`（state 映射：0→Stable, 1→Start, 2→Move, 其他→Stop），`ToWasmTouchUpdate → TouchUpdateEvent`（含键盘修饰符解包）。

### 鼠标事件

| 结构体 | 字段 |
|--------|------|
| `WMouse` | `x`, `y`, `modifiers`, `button`, `time` |
| `ToWasmMouseDown` | `mouse: WMouse` |
| `ToWasmMouseMove` | `was_out`, `mouse` |
| `ToWasmMouseUp` | `mouse` |

按钮映射函数 `wmouse_to_mouse_button`：0→PRIMARY, 1→MIDDLE, 2→SECONDARY, 其他→`from_raw_button`。

每个鼠标事件均转换为相应的 Makepad 事件类型（`MouseDownEvent`、`MouseMoveEvent`、`MouseUpEvent`），均包含 DPI 坐标、窗口 ID（id_zero）、修饰符和时间戳。

### 滚动事件

`ToWasmScroll` → `ScrollEvent`：含 `scroll_x/y`、位置、修饰符、`is_mouse`（固定 true）。

### 键盘事件

`web_to_key_code` 函数：将 Web keyCode（数值）映射到 `KeyCode` 枚举（约 70+ 个映射），覆盖字母、数字、功能键、方向键、小键盘、符号键等。

`WKey` → `KeyEvent` 转换：包含 `char_code`、`key_code`、`modifiers`、`time`、`is_repeat`。

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `WKey` | `char_code`, `key_code`, `modifiers`, `time`, `is_repeat` | 按键描述 |
| `ToWasmKeyDown` | `key: WKey` | 按键按下 |
| `ToWasmKeyUp` | `key: WKey` | 按键释放 |
| `ToWasmTextInput` | `was_paste`, `replace_last`, `input` | 文本输入 |
| `ToWasmTextCopy` | — | 文本复制请求 |

`ToWasmTextInput → TextInputEvent`：保留 `was_paste`、`replace_last` 和文本内容。

### 焦点事件

- `ToWasmWindowGotFocus` — 窗口获得焦点
- `ToWasmWindowLostFocus` — 窗口失去焦点

### HTTP 网络响应

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `ToWasmHTTPResponse` | `request_id_lo/hi`, `metadata_id_lo/hi`, `status`, `headers`, `body` | HTTP 响应 |
| `ToWasmHttpRequestError` | `request_id_lo/hi`, `metadata_id_lo/hi`, `error` | HTTP 请求错误 |
| `ToWasmHttpResponseProgress` | `request_id_lo/hi`, `metadata_id_lo/hi`, `loaded`, `total` | 下载进度 |
| `ToWasmHttpUploadProgress` | `request_id_lo/hi`, `loaded`, `total` | 上传进度 |

### 权限

`ToWasmPermissionResult` — `permission`（字符串 "microphone"/"camera"）、`request_id`、`status`（0=NotDetermined,1=Granted,2=DeniedCanRetry,3=DeniedPermanent）。

### MIDI

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `ToWasmMidiInputData` | `uid`, `data: u32` | MIDI 输入数据 |
| `WMidiPortInfo` | `name`, `uid`, `is_output` | MIDI 端口描述 |
| `ToWasmMidiPortList` | `ports: Vec<WMidiPortInfo>` | MIDI 端口列表 |

### 音频

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `WAudioDevice` | `web_device_id`, `label`, `is_output` | 音频设备描述 |
| `ToWasmAudioDeviceList` | `devices: Vec<WAudioDevice>` | 音频设备列表 |

### 视频播放

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `ToWasmVideoPlaybackPrepared` | `video_id_lo/hi`, `video_width`, `video_height`, `duration_lo/hi` | 视频加载就绪 |
| `ToWasmVideoTextureUpdated` | `video_id_lo/hi`, `current_position_lo/hi` | 视频新帧可用 |
| `ToWasmVideoPlaybackCompleted` | `video_id_lo/hi` | 视频播放完成 |
| `ToWasmVideoPlaybackResourcesReleased` | `video_id_lo/hi` | 视频资源释放 |

## 键盘修饰符工具

`unpack_key_modifier(modifiers: u32) → KeyModifiers` — 将位掩码解包为 shift(bit0)、control(bit1)、alt(bit2)、logo(bit3)。

## 注释部分

原有的 WebSocket 消息（`ToWasmWebSocketOpen/Close/Error/String/Binary`）已注释掉，表明 WebSocket 处理已迁移到替代实现（`web_network.rs` 中的 `WasmNetworkShimBackend` 和 `web_socket.rs` 中的 `OsWebSocket`）。
