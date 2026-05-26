# media_foundation.rs — Media Foundation 摄像头访问

**文件路径**: platform/src/os/windows/media_foundation.rs (614行)
**核心用途**: 通过 Windows Media Foundation API 枚举和访问摄像头设备，支持异步帧回调、多种像素格式转换和设备变更通知。

## 常量

- `MFVideoFormat_GRAY: GUID = 0x3030_3859_0000_0010_8000_00aa00389b71` — 自定义灰度格式 GUID

## 结构体

### `MfInput`
```rust
struct MfInput {
    destroy_after_update: bool,        // 标记是否在更新后移除
    symlink: String,                   // 设备符号链接
    active_format: Option<VideoFormatId>, // 当前激活的格式
    desc: VideoInputDesc,              // 设备描述
    reader_callback: IMFSourceReaderCallback, // 异步帧回调
    source_reader: IMFSourceReader,    // 源读取器
    media_types: Vec<MfMediaType>,     // 支持的媒体类型
}
```

### `MfMediaType`
```rust
struct MfMediaType {
    format_id: VideoFormatId,
    media_type: IMFMediaType,
}
```

### `MediaFoundationAccess`
```rust
pub struct MediaFoundationAccess {
    pub video_input_cb: [Arc<Mutex<Option<VideoInputFn>>>; MAX_VIDEO_DEVICE_INDEX],
    pub camera_frame_input_cb: [Arc<Mutex<Option<CameraFrameInputFn>>>; MAX_VIDEO_DEVICE_INDEX],
    pub video_output_cb: [Arc<Mutex<Option<VideoOutputFn>>>; MAX_VIDEO_DEVICE_INDEX],
    pub video_encoder_config: [Arc<Mutex<Option<VideoEncoderConfig>>>; MAX_VIDEO_DEVICE_INDEX],
    video_encoder: [Arc<Mutex<Option<VideoEncoder>>>; MAX_VIDEO_DEVICE_INDEX],
    inputs: Vec<MfInput>,
    _enumerator: IMMDeviceEnumerator,
    _change_listener: IMMNotificationClient,
}
```

### `SourceReaderConfig`
```rust
struct SourceReaderConfig {
    video_format: VideoFormat,
    callback: Arc<Mutex<Option<VideoInputFn>>>,
    frame_callback: Arc<Mutex<Option<CameraFrameInputFn>>>,
    video_encoder: Arc<Mutex<Option<VideoEncoder>>>,
}
```

### `SourceReaderCallback`
通过 `implement_com!` 注册 `IMFSourceReaderCallback`，异步接收摄像头帧。

### `MediaFoundationChangeListener`
通过 `implement_com!` 注册 `IMMNotificationClient`，监听音频/视频设备变更。

## 关键函数

### `camera_frame_from_media_buffer`
将 MF 媒体缓冲区转换为 `CameraFrameRef`：
- 支持 `NV12`: 分离 Y 平面和 UV 平面，从行步幅计算平面偏移
- 支持 `YUY2`: 打包格式，计算行步幅
- 支持 `MJPEG`: 直接传递 JPEG 数据

### `MediaFoundationAccess` 方法

| 方法 | 描述 |
|------|------|
| `new(change_signal)` | 创建并初始化。创建 `MMDeviceEnumerator`，注册 `MediaFoundationChangeListener` 接收设备变更通知 |
| `use_video_input(inputs)` | 激活指定摄像头和格式。自动配置视频编码器（如果启用了输出回调） |
| `get_updated_descs()` | 枚举所有接入的摄像头设备，查询支持的媒体类型和分辨率 |

### `MfInput::activate`
1. 设置 `SourceReaderConfig`（格式、回调、编码器）
2. 调用 `SetCurrentMediaType` 配置格式
3. 调用 `ReadSample` 触发第一帧

### `get_updated_descs` 设备枚举流程
1. `MFCreateAttributes` → `MF_DEVSOURCE_ATTRIBUTE_SOURCE_TYPE_VIDCAP_GUID`
2. `MFEnumDeviceSources` 枚举所有视频捕获设备
3. 对每个设备：获取名称和符号链接，创建 IMFSourceReader
4. 枚举原生媒体类型 (`GetNativeMediaType`)：解析帧大小、帧率、像素格式
5. 生成 `VideoFormatId`（基于 "width height fps PixelFormat" 字符串的 LiveId）

## IMFSourceReaderCallback_Impl

### `OnReadSample`
异步帧接收核心：
1. 从 `IMFSample` 获取 `IMFMediaBuffer`，`Lock` 获得原始数据指针
2. 调用 `camera_frame_from_media_buffer` 转换帧
3. 触发 `frame_callback`（原始帧数据）和 `video_input_cb`（根据格式提供 `U8` 或 `U32` 数据）
4. 如果编码器已配置，将帧推送给编码器
5. `Unlock` 释放缓冲区
6. 队列下一次 `ReadSample` 调用（持续采集）

### 像素格式支持
| MF 格式 | Makepad PixelFormat |
|---------|-------------------|
| `MFVideoFormat_RGB24` | `RGB24` |
| `MFVideoFormat_YUY2` | `YUY2` |
| `MFVideoFormat_NV12` | `NV12` |
| `MFVideoFormat_GRAY` | `GRAY` |
| `MFVideoFormat_MJPG` | `MJPEG` |
| 其他 | `Unsupported(guid.data1)` |

## IMMNotificationClient_Impl

`MediaFoundationChangeListener` 监听 `OnDeviceStateChanged` 事件，触发 `change_signal.set()` 通知主线程重新枚举设备。

## 平台集成

- 通过 `CxWindowsMedia::media_foundation()` 延迟初始化
- `Cx::handle_media_signals()` 检查变更信号并触发 `Event::VideoInputs`
- `CxMediaApi` 的 `use_video_input` / `video_input_box` / `camera_frame_input_box` 方法桥接到 Makepad 平台抽象
