# linux_video_player.rs — 统一视频播放器枚举

**文件路径**: `platform/src/os/linux/linux_video_player.rs` (243 行)
**核心功能**: 将 GStreamer 原生播放器、软件 rav1d 回退播放器和 V4L2 摄像头捕获统一为单一 `LinuxVideoPlayer` 枚举。

## 主要类型

### `YuvTextureSet`
YUV 三纹理集合，包含 `tex_y` / `tex_u` / `tex_v` `Texture` 对象和对应的 `YuvTextureIds`。

### `LinuxVideoPlayer`
视频播放器枚举，三种变体：

```rust
pub enum LinuxVideoPlayer {
    GStreamer { player: GStreamerVideoPlayer, yuv: Option<YuvTextureSet> },
    Software { player: PlaybackSessionHandle, yuv: YuvTextureSet, yuv_matrix: f32 },
    Camera(V4l2CameraPlayer),
}
```

## 方法

所有方法均委托给内部播放器变体：

### 生命周期
- `check_prepared` — 检查播放器准备状态
- `poll_frame(gl, textures)` — 从播放器拉取帧并上传 GL 纹理
- `check_eos` — 检查流结束
- `cleanup` — 释放资源

### 播放控制
`play` / `pause` / `resume` / `mute` / `unmute` / `seek_to` / `set_volume` / `set_playback_rate`

### 状态查询
`video_id` / `is_active` / `is_software_mode` / `is_camera_mode` / `is_yuv_mode` / `yuv_texture_set` / `yuv_matrix` / `current_position_ms` / `seekable_ranges` / `buffered_ranges`

## 实现细节

- `Software` 变体在 `poll_frame` 中通过 `take_yuv_frame` 获取解码后的 YUV 平面，然后调用 `upload_yuv_to_gl` 上传
- `GStreamer` 变体中 `yuv` 是可选的（仅在使用 I420 降级路径时设置）
- `Camera` 变体不可 seek，不支持暂停/音量控制
- `yuv_matrix` 返回 BT.601/BT.709 色彩矩阵选择（Camera 默认 BT.601 = 1.0）
