# linux_video_playback.rs — GStreamer 视频播放器

**文件路径**: `platform/src/os/linux/linux_video_playback.rs` (1101 行)
**核心功能**: 基于 GStreamer 的视频播放器实现，支持网络 URL、本地文件和内存数据的解码播放，包含零拷贝 GL 内存路径和系统内存回退路径。

## MIME 类型检测

`can_play_type(mime)` — 基于硬编码表返回 `"probably"`/`"maybe"`/`""`：
- `video/mp4` / `video/webm` / `video/ogg` → `"probably"`
- `video/x-matroska` / `audio/mp4` / `audio/mpeg` 等 → 常见格式标记

## 枚举

### `YuvTextureIds`
YUV 三平面的纹理 ID 集合（`tex_y_id`, `tex_u_id`, `tex_v_id`）。

### `VideoCapsProfile`
GStreamer caps 协商的降级链：
- `GlMemoryRgba2D` → `GlMemoryRgba` → `SystemI420` → `SystemRgba`

## 主要类型

### `GStreamerVideoPlayer`
视频播放器状态机，包含：
- `gst` — GStreamer 函数表指针
- `pipeline` / `video_sink` / `bus` — GStreamer 管道元素
- `video_id` / `texture_id` / `yuv_ids` — 纹理标识
- `caps_profile` — 当前使用的 caps 配置
- `retained_gl_sample` — 保留的 GLMemory 样本（零拷贝路径）
- `pixel_buf` — 回退路径的临时行打包缓冲区

## 关键方法

### 构建与销毁
- `new` / `new_audio_only` — 创建视频或纯音频播放器
- `build_pipeline` — 构建 GStreamer 管道（`playbin` + `appsink` 或 `fakesink`）
- `destroy_pipeline` — 销毁管道并释放资源

### 生命周期
- `check_prepared` — 检查管道是否完成预滚（preroll），返回 `PlaybackPrepared` 或错误
- `poll_frame` — 从 appsink 拉取解码帧并上传到 GL 纹理
- `check_eos` — 检查是否到达流末尾

### Caps 降级
当 caps 协商失败时自动降级尝试下一个配置，从 `GlMemoryRgba2D` 到 `SystemRgba`。

### 播放控制
`play` / `pause` / `resume` / `mute` / `unmute` / `seek_to` / `set_volume` / `set_playback_rate`

### 状态查询
`current_position_ms` / `seekable_ranges` / `buffered_ranges` / `is_active` / `is_yuv_mode`

## 帧上传路径

### 零拷贝路径（GLMemory）
如果 GStreamer 检测到 `GstGLMemory`：
1. 从 `GstMemory` 获取 OpenGL 纹理 ID
2. 直接将纹理引用设到 `CxTexturePool`（标记 `gl_texture_owned = false`）
3. 保留 `GstSample` 防止纹理被回收

### 系统内存路径
1. `I420` 路径：直接从映射缓冲区切片上传 Y/U/V 平面
2. `RGBA` 路径：处理可能的行跨度（stride）不匹配，必要时通过 `pixel_buf` 重新打包
3. 使用 `glTexImage2D`（新分配）或 `glTexSubImage2D`（更新）

## 清理

`cleanup` 方法释放保留的 GL 样本、销毁管道、删除临时文件。
