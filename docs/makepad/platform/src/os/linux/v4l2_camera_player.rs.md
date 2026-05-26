# v4l2_camera_player.rs

One-liner (EN): V4L2 camera playback integration — captures frames from a V4L2 camera device and uploads YUV planes to OpenGL textures for rendering.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/v4l2_camera_player.rs` (160 行)
- **核心作用**: 将 V4L2 摄像头捕获的帧数据作为视频播放源接入 Makepad 的渲染管线。通过帧池（`CameraFramePool`）缓冲 YUV 帧，并使用 `upload_i420_slices_to_gl` 将 Y/U/V 三个平面上传到 OpenGL 纹理。

## 关键类型

### `V4l2CameraPlayer`

| 字段 | 类型 | 说明 |
|------|------|------|
| `video_id` | `LiveId` | 视频标识符 |
| `tex_y_id / tex_u_id / tex_v_id` | `TextureId` | Y/U/V 平面对应的 OpenGL 纹理 ID |
| `_input_id` | `VideoInputId` | 摄像头输入 ID |
| `_format_id` | `VideoFormatId` | 视频格式 ID |
| `width / height` | `u32` | 当前帧尺寸 |
| `active` | `bool` | 播放器是否激活 |
| `prepared / prepare_notified` | `bool` | 准备就绪状态跟踪 |
| `frame_pool` | `Arc<Mutex<CameraFramePool>>` | 线程安全的帧池 |
| `camera_access` | `Option<Arc<Mutex<V4l2CameraAccess>>>` | 摄像头控制接口 |

## 关键方法

| 方法 | 说明 |
|------|------|
| `new(video_id, tex_y_id, tex_u_id, tex_v_id, input_id, format_id, camera_access)` | 创建播放器，注册帧回调到摄像头捕获管线 |
| `is_active() -> bool` | 检查播放器是否仍处于激活状态 |
| `check_prepared() -> Option<Result<PlaybackPrepared, String>>` | 检查第一帧是否已到达，返回媒体准备信息（宽高、track 列表） |
| `poll_frame(gl, textures) -> bool` | 从帧池获取最新帧，上传 Y/U/V 纹理数据；返回是否有新帧 |
| `cleanup()` | 清理摄像头回调引用和帧池 |

## 实现细节

### 帧流架构

```
V4L2 Camera (捕获线程)
  │
  │ CameraFrameInputFn 回调
  │   convert_to_i420()
  ▼
CameraFramePool (Arc<Mutex<>>)
  │   publish_latest()
  │
  ▼
V4l2CameraPlayer::poll_frame() (渲染线程)
  │   take_latest()
  │   upload_i420_slices_to_gl()
  ▼
GL Textures (tex_y_id, tex_u_id, tex_v_id)
```

- **帧池**: 使用 4 帧容量的 `CameraFramePool` 作为生产-消费者缓冲区
- **格式转换**: 在回调中将原始帧转换为 I420 平面格式，再通过 `poll_frame` 上传到 GL
- **纹理上传**: 使用 `upload_i420_slices_to_gl` 函数，将三个 Y/U/V 平面分别上传到对应的 GL 纹理
- **生命周期管理**: `cleanup()` 清空摄像头回调，`Drop` 自动调用 `cleanup()`
- **准备通知**: `check_prepared()` 只发送一次 `PlaybackPrepared`，基于第一帧到达
