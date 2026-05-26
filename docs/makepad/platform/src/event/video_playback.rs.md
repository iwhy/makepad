# `video_playback.rs` — 视频播放事件系统

## 概述

该文件定义了视频播放全生命周期中的各类事件类型，从资源准备、纹理更新、播放完成到解码错误。同时包含 YUV 元数据系统和视频源类型枚举。

---

## 事件类型

### `VideoPlaybackPreparedEvent`
视频解码器初始化完成，可以开始播放：
- `video_id: LiveId` — 视频实例标识
- `video_width: u32` / `video_height: u32` — 视频原始分辨率
- `duration: u128` — 总时长（毫秒）
- `is_seekable: bool` — 是否支持 seek 操作
- `video_tracks: Vec<String>` — 视频轨道描述（仅音源时为空）
- `audio_tracks: Vec<String>` — 音频轨道描述

### `VideoTextureUpdatedEvent`
视频帧纹理已更新：
- `video_id: LiveId`
- `current_position_ms: u128` — 当前播放位置（毫秒）
- `yuv: VideoYuvMetadata` — YUV 色彩空间元信息

### `VideoPlaybackCompletedEvent`
- `video_id: LiveId` — 播放完成通知

### `VideoPlaybackResourcesReleasedEvent`
- `video_id: LiveId` — 播放资源已释放

### `VideoDecodingErrorEvent`
- `video_id: LiveId`
- `error: String` — 错误描述

### `TextureHandleReadyEvent`
- `texture_id: TextureId`
- `handle: u32` — GPU 纹理句柄

### `VideoYuvTexturesReady`
YUV 平面纹理已分配（由平台后端触发的内部事件）：
- `video_id: LiveId`
- `tex_y: Texture` — Y 平面纹理
- `tex_u: Texture` — U 平面纹理
- `tex_v: Texture` — V 平面纹理

Video widget 使用此事件将纹理绑定到着色器插槽。

### `VideoSeekableRangesEvent`
- `video_id: LiveId`
- `ranges: Vec<(f64, f64)>` — 可 seek 的时间范围列表，每个元素为 `(start_seconds, end_seconds)`

### `VideoBufferedRangesEvent`
- `video_id: LiveId`
- `ranges: Vec<(f64, f64)>` — 已缓冲（下载/解码完成）的时间范围列表

---

## `VideoYuvMetadata`

YUV 色彩空间元数据：

| 字段 | 类型 | 描述 |
|------|------|------|
| `enabled` | `bool` | 是否启用 YUV 纹理（而非外部 RGB） |
| `matrix` | `f32` | 颜色矩阵：0.0=BT.709, 1.0=BT.601, 2.0=BT.2020 |
| `biplanar` | `bool` | UV 是否在单个 RG8 纹理中（NV12 双平面） |
| `rotation_steps` | `f32` | YUV 顺时针旋转步数（0/1/2/3） |

### 方法
- `disabled()` — 创建禁用的默认元数据（`enabled = false`）
- `shader_enabled() -> f32` — 布尔转着色器 `1.0/0.0`
- `shader_biplanar() -> f32` — 布尔转着色器 `1.0/0.0`

---

## `CameraPreviewMode` 枚举
- `Texture` — 纯纹理模式
- `Native` — 原生预览
- `Auto` — 自动选择

---

## `VideoSource` 枚举

视频来源类型：
- `InMemory(Rc<Vec<u8>>)` — 内存数据
- `Network(String)` — 网络 URL
- `Filesystem(String)` — 文件系统路径
- `Camera(VideoInputId, VideoFormatId)` — 摄像头输入
- `PlaybackSession(MediaPlaybackSessionId)` — 回放会话
- `Session(VideoFrameSessionId)` — 视频帧会话

### `VideoSource::is_session() -> bool`
判断是否为会话类型（`PlaybackSession` 或 `Session`）。
