# av_capture.rs — 摄像头捕获

**文件路径:** `platform/src/os/apple/av_capture.rs`

**核心目的:** 使用 AVFoundation 实现摄像头视频捕获。提供 `AvCaptureSession` 管理摄像头输入、预览和帧捕获，支持将视频帧直接输出为 Metal 纹理。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `AvCaptureSession` | 摄像头捕获会话管理器 |
| `AvCaptureDeviceInfo` | 摄像头设备信息（ID、名称、位置、格式等） |
| `AvCaptureFormat` | 捕获格式描述（分辨率、帧率、像素格式） |
| `AvCapturePhoto` | 静态照片数据 |
| `AvCaptureVideoFrame` | 视频帧数据，包含 Metal 纹理引用 |
| `CameraPosition` | 摄像头位置枚举（前置/后置/外部） |

**关键方法:**
- `AvCaptureSession::new()` — 创建捕获会话：
  - 请求摄像头权限（`AVMediaTypeVideo`）
  - 创建 `AVCaptureSession`
  - 初始化 `CVMetalTextureCache` 用于 GPU 纹理转换
- `AvCaptureSession::start_session()` — 启动捕获
- `AvCaptureSession::stop_session()` — 停止捕获
- `AvCaptureSession::switch_camera()` — 切换前后摄像头
- `AvCaptureSession::set_camera(pos)` — 设置指定位置摄像头
- `AvCaptureSession::get_available_devices()` — 获取可用摄像头列表
- `AvCaptureSession::set_active_format(format)` — 设置捕获格式（分辨率/帧率）
- `AvCaptureSession::capture_photo(cb)` — 捕获静态照片

**帧处理流水线:**
1. `AVCaptureDeviceInput` 从摄像头捕获帧
2. `AVCaptureVideoDataOutput` 输出 `CMSampleBuffer`
3. `CMSampleBufferGetImageBuffer` 提取 `CVPixelBuffer`
4. `CVMetalTextureCacheCreateTextureFromImage` 创建 Metal 纹理
5. YUV 或 BGRA 纹理通过回调传递给应用

**`AvCaptureDeviceInfo` 字段:**
- `unique_id: String` — 设备唯一标识符
- `localized_name: String` — 本地化设备名
- `position: CameraPosition` — 位置（前置/后置/外部）
- `formats: Vec<AvCaptureFormat>` — 支持的格式列表
- `has_torch: bool` — 是否支持闪光灯
- `has_flash: bool` — 是否支持手电筒

**`AvCaptureFormat` 字段:**
- `width: u32`, `height: u32` — 分辨率
- `frame_rate: f64` — 最大帧率
- `pixel_format: u32` — 像素格式（kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange 等）
- `is_hdr: bool` — 是否支持 HDR

**实现细节:**
- 权限处理：通过 `AVCaptureDevice_requestAccessForMediaType` 异步请求
- 使用 `AVCaptureDeviceDiscoverySession` 枚举设备
- 支持自动对焦、自动白平衡、自动曝光
- 帧率可通过 `AVCaptureDevice_setActiveVideoMinFrameDuration` 控制
- 照片捕获使用 `AVCapturePhotoOutput`
- 闪光灯/手电筒通过 `AVCaptureDevice_setTorchMode` 控制
- 支持缩放（通过 `videoZoomFactor`）
- 错误恢复：设备断开时自动重连

**平台集成:** macOS 和 iOS/tvOS，使用 AVFoundation、CoreVideo、Metal 框架
