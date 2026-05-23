# `playback_session.rs` — 播放会话管理

## 概述

管理自定义媒体播放会话的注册、查找和音频混音的生命周期。外部播放器（如视频解码插件、自定义流媒体实现）通过此模块向 Makepad 音频管线注册会话，实现音视频输出的统一混音。

## 核心类型

### `MediaPlaybackSessionId(u64)`
会话的不透明句柄，由全局原子计数器 `MEDIA_PLAYBACK_SESSION_IDS` 自增分配（起始为 1）。实现了 `Clone`、`Copy`、`PartialEq`、`Eq`、`Hash`。

## 全局状态管理

使用三个 `OnceLock` 包装的 `Mutex<HashMap<…>>` 全局变量实现线程安全的会话存储：

- **`MEDIA_PLAYBACK_SESSIONS`**：存放尚未被消费的注册会话，键为 `MediaPlaybackSessionId`，值为 `Box<dyn MediaPlaybackSession + Send>`。
- **`ACTIVE_MEDIA_AUDIO`**：存放已激活的音频会话，键为 `MediaPlaybackSessionId`，值为 `Arc<Mutex<Box<dyn MediaPlaybackSession + Send>>>`，支持多线程并发读取。

## 公共函数

### `register_media_playback_session(session) -> MediaPlaybackSessionId`
注册一个新的播放会话。原子递增 ID 计数器，将 session 插入全局 sessions map，返回分配的唯一 ID。通常在创建解码器实例时调用。

### `unregister_media_playback_session(id) -> Option<Box<…>>`
从注册表中移除一个尚未被消费的会话，返回其所有权。若 ID 不存在则返回 `None`。用于清理。

### `take_registered_media_playback_session(id) -> Option<Box<…>>`
`#[doc(hidden)]` 标记的内部函数，直接委托给 `unregister_media_playback_session`。表示"拿走"已注册会话的所有权。

### `register_active_media_audio(id, session)`
将会话加入活跃音频混音列表。内部的 `SharedPlaybackSession`（`Arc<Mutex<…>>`）允许多个组件安全共享同一会话：一个线程填入音频数据，另一个线程在混音时读取。

### `unregister_active_media_audio(id)`
将会话从活跃音频混音列表中移除。通常在会话播放完毕或用户关闭媒体时调用。

### `mix_active_media_audio(info, output)`
将当前所有活跃会话的音频混入一个输出缓冲。实现逻辑：
1. 将 `output` 缓冲清零。
2. 获取所有活跃会话的快照（`Vec<SharedPlaybackSession>`，短暂持有锁后释放）。
3. 遍历每个会话，锁定后调用 `fill_audio_output(info, output)` 将各会话的 PDM（音频样本）叠加（sum）到输出缓冲中。多个会话的样本以加法混音（additive mixing）方式合并。

## 单元测试

### `StubSession`
模拟会话桩，实现 `MediaPlaybackSession` trait。在 `fill_audio_output` 中将输出缓冲的第一个采样加 0.5，辅助验证混音行为。

### `registry_roundtrip`
验证注册→取出→二次取出的完整流程：注册后能取出，取出后再次取应返回 `None`。

### `active_audio_mix_calls_registered_sessions`
验证音频混音：注册一个活跃会话→调用 `mix_active_media_audio`→验证输出缓冲中有数据→反注册。
