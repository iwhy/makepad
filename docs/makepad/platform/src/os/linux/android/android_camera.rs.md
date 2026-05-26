# Android 摄像头实现

## 概述

`android_camera.rs` 基于 Camera2 NDK API（`acamera_sys`）实现了 Android 摄像头预览、视频帧采集和视频编码功能。它管理摄像头设备枚举、流配置、捕获会话生命周期以及编码器集成。

## 核心类型

### `AndroidCameraDevice`
摄像头设备信息：
- `camera_id_str` — Camera2 ID（CString）
- `desc` — `VideoInputDesc`（ID、名称、格式列表）
- `sensor_orientation_degrees` — 传感器方向（0/90/180/270）

### `CameraStreamKey`
流唯一键（`(input_id, format_id)`）。

### `StreamDispatch`
流分发配置，包含各类型的回调列表：
- `video_input_cbs` — `VideoInputFn` 回调
- `frame_input_cbs` — `CameraFrameInputFn` 回调
- `preview_frame_input_cbs` — 预览帧回调
- `preview_hardware_buffer_input_cbs` — AHardwareBuffer 预览回调
- `encoders` — `VideoEncoder` 引用

### `AndroidImageReaderMode`
图像读取器模式：
- `CpuReadable` — CPU 可读模式（标准帧回调）
- `HardwareBufferYuv` — AHardwareBuffer YUV 模式（零拷贝）

### `AndroidCameraHardwareBufferFrame`
AHardwareBuffer 帧（含 buffer 指针、时间戳、尺寸）。`Drop` 实现自动调用 `AHardwareBuffer_release`。

### `CameraHardwareBufferInputFn`
AHardwareBuffer 帧回调类型别名。

### `PreviewSubscription`
预览订阅记录（VideoId → 流键 + 帧回调 + 预览窗口）。

### `CameraStreamNode`
流节点状态（camera_id、格式、分发、会话、预览窗口）。

### `AndroidCaptureSession`
完整的 Camera2 捕获会话，包含：

| 字段 | 说明 |
|------|------|
| `capture_session` | `ACameraCaptureSession` |
| `output_container` | 输出容器 |
| `image_output` / `preview_output` | 输出 |
| `camera_device` | `ACameraDevice` |
| `image_target` / `preview_target` | 输出目标 |
| `image_window` / `preview_window` | `ANativeWindow` |
| `image_reader` | `AImageReader` |
| `capture_request` | `ACaptureRequest` |
| `capture_context` | `AndroidCaptureContext` |

### `AndroidCaptureContext`
捕获上下文（分发、格式、存活标志、读取器模式）。

### `AndroidCameraAccess`
摄像头访问管理器：
- `video_input_cb` / `camera_frame_input_cb` — 输入回调数组（固定大小 `MAX_VIDEO_DEVICE_INDEX`）
- `video_output_cb` / `video_encoder_config` / `video_encoder` — 编码器相关
- `manager` — `ACameraManager`
- `devices` — 已知设备列表
- `streams` — `HashMap<CameraStreamKey, CameraStreamNode>`
- `slot_streams` — 当前活动流的槽位映射
- `preview_subscriptions` — 预览订阅
- `active_inputs` — 当前活动的输入/格式对

## 关键方法

### `AndroidCameraAccess`

#### `new(change_signal)`
创建管理器，调用 `ACameraManager_create()`。

#### `get_updated_descs() -> Vec<VideoInputDesc>`
枚举摄像头设备：
1. `ACameraManager_getCameraIdList` 获取摄像头列表
2. 对每个摄像头查询 `ACAMERA_LENS_FACING`（Front/Back/External）
3. 查询 `ACAMERA_SENSOR_ORIENTATION`（旋转角度）
4. 解析 `ACAMERA_SCALER_AVAILABLE_STREAM_CONFIGURATIONS` 获取可用格式
5. 过滤 YUV420 和 JPEG 格式
6. `reconcile_streams()` 同步流状态

#### `use_video_input(inputs)`
激活视频输入流：
1. 更新 `slot_streams` 映射
2. 调用 `refresh_slot_encoder(index)` 刷新编码器配置
3. `reconcile_streams()` 启动/停止流

#### `register_preview` / `register_preview_hardware_buffer`
注册预览帧回调或 AHardwareBuffer 回调。

#### `update_preview_window` / `unregister_preview`
管理预览窗口。

#### `configure_video_encoder`
配置视频编码器（支持 Camera 源和 Texture 源）。

#### `video_encoder_push_frame` / `video_encoder_request_keyframe`
推送帧 / 请求关键帧。

#### `video_encoder_capture_texture_frame`
从 GL 纹理捕获帧进行编码：
1. 绑定纹理到 FBO
2. `glReadPixels` 读取 RGBA
3. `convert_rgba_8888_to_i420` 转换格式
4. 推送给编码器

#### `reconcile_streams()`
核心流同步逻辑：
1. 计算所需流（从 `slot_streams` 和 `preview_subscriptions`）
2. 移除不再需要的流（调用 `session.stop()`）
3. 为新增流创建 `CameraStreamNode`
4. 对每个流调用 `restart_stream_if_needed`

### `AndroidCaptureSession`

#### `start(dispatch, manager, camera_id, format, preview_window, needs_image_reader) -> Option<Self>`
创建捕获会话：
1. 检测 `AndroidImageReaderMode`
2. 打开摄像头设备（`ACameraManager_openCamera`）
3. 创建捕获请求（`TEMPLATE_PREVIEW`）
4. 创建 `AImageReader`（根据格式选择 `AImageReader_new` 或 `AImageReader_newWithUsage`）
5. 注册图像可用回调（`AImageReader_ImageListener`）
6. 创建预览目标（如果提供 `preview_window`）
7. 构建输出容器
8. 创建捕获会话
9. 设置重复请求（`ACameraCaptureSession_setRepeatingRequest`）

图像可用回调 `image_on_image_available` 处理：
- **HardwareBufferYuv 模式**：获取 AHardwareBuffer，分发给硬件缓冲回调
- **MJPEG 格式**：读取平面数据，分发 `CameraFrameRef` 和 `VideoBufferRef`
- **YUV420 格式**：读取 Y/U/V 三平面数据，组装 `CameraFrameRef`（I420 布局），分发给帧回调和编码器；同步为 `VideoInputFn` 回调生成 packed YUV 数据

#### `stop(self)`
停止捕获会话：
1. 设置 `alive = false`（阻止后续回调）
2. 移除图像监听器
3. 停止重复请求
4. 关闭会话、释放所有输出/目标/窗口/读取器资源
5. 关闭设备

### 静态回调函数

| 回调 | 用途 |
|------|------|
| `device_on_disconnected/error` | 设备状态 |
| `session_on_closed/ready/active` | 会话状态 |
| `capture_on_started/progressed/completed/failed/sequence/aborted/buffer_lost` | 捕获过程 |
| `image_on_image_available` | 帧数据分发（核心逻辑） |

## 实现说明

- 所有摄像头 NDK 操作在 `unsafe` 块中执行。
- `AndroidCaptureSession` 的帧回调使用 `Context` 指针传递 `AndroidCaptureContext`。
- `StreamDispatch` 保存 `Arc<Mutex<Option<XxxFn>>>` 引用，多个流节点可共享同一槽位的回调。
- `refresh_slot_encoder` 在编码器配置变化或流变化时自动启动/停止编码器。
- `VideoEncoder::start` 创建 `VideoEncoder` 并通过 `Box<dyn FnMut>` 输出编码数据。
- 此文件不直接涉及 Android 摄像头预览 Surface（Java 侧 SurfaceView 管理），该功能在 `android_camera_player.rs` 中实现。
