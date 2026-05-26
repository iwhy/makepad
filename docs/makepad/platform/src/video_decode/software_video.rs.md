# `software_video.rs` — 软件回放会话句柄

## 概述
该文件提供 `PlaybackSessionHandle`，作为 `MediaPlaybackSession` trait 的 Rust 侧封装。它统一管理媒体插件的创建、帧轮询、播放控制命令，并在插件不可用时提供降级路径（返回错误而非崩溃）。

---

## `PlaybackSessionHandle`

### 字段
- **`video_id: LiveId`**：视频资源的 LiveId 标识。
- **`texture_id: TextureId`**：解码帧上传到的 GPU 纹理 ID。
- **`inner: Option<Box<dyn MediaPlaybackSession>>`**：底层媒体插件会话，为 `None` 表示创建失败。
- **`failed: Option<String>`**：保存创建失败原因，延迟到首次操作时上报错误。

### 构造流程 `new(video_id, texture_id, source, autoplay, is_looping)`

1. 调用 `media_plugin()` 获取全局媒体插件实例。
2. 若插件存在则调用 `create_playback_session()` 创建会话；成功则 `inner = Some(session)`，失败则 `failed = Some(err)`。
3. 若无媒体插件安装，`failed` 设置为 `"no media plugin installed"`。
4. 无论成功或失败，video_id 和 texture_id 始终保留以支持后续查询。

### 方法

**状态查询与帧获取：**
- **`check_prepared()`**：返回 `Option<Result<PlaybackPrepared, String>>`。若插件存在则委派 `inner.check_prepared()`；若插件创建失败则通过 `failed.take()` 返回一次性的错误信息。
- **`poll_frame()`**：驱动插件解码下一帧，返回 `bool` 表示是否有新帧可用。
- **`take_yuv_frame()`**：获取最新解码帧的 YUV 平面数据。返回 `Option<YuvPlaneData>`。
- **`check_eos()`**：检查是否到达流末尾。
- **`decode_error()`**：若 `inner` 存在则返回 `Ok(())`，否则返回 `Err(VideoDecodeError::UnsupportedCodec)`。

**播放控制：**
- **`play()` / `pause()` / `resume()`**：播放/暂停/恢复，委派给底层 session。
- **`is_playing()`**：查询播放状态。
- **`seek_to(position_ms)`**：跳转到指定毫秒位置。
- **`set_volume(volume)`**：设置音量（0.0~1.0）。
- **`set_playback_rate(rate)`**：设置播放速率。
- **`mute()` / `unmute()`**：静音/取消静音。

**媒体信息：**
- **`current_position_ms()`**：返回当前播放位置（毫秒）。
- **`seekable_ranges()`**：返回可跳转的时间范围列表。
- **`buffered_ranges()`**：返回已缓冲的时间范围列表。
- **`is_active()`**：检查会话是否还处于有效状态。

**资源管理：**
- **`cleanup()`**：触发清理，释放解码器和后端资源。
