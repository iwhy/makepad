# `video_session.rs` — 视频帧会话注册与生命周期管理

## 概述
该文件定义了 Makepad 泛用视频路径中应用层解码帧会话的 trait 和注册表。它允许应用（如 MSE MediaSource 播放器）将其自有的解码帧通道注册为全局可查找的会话，供 `Video` widget 在播放绑定阶段按 ID 获取。

---

## 核心枚举

### `VideoSessionState`
视频帧会话的生命周期状态：
- **`Connecting`**：初始化中，尚未产出可显示帧。
- **`Active`**：会话活跃，可能产生新的解码帧。
- **`Ended`**：会话正常结束。
- **`Error(String)`**：会话失败，附带错误信息。

---

## Trait 定义

### `VideoFrameSession`
应用端需实现的 trait，要求 `Send` 以支持跨线程传递：
- **`take_frames(&mut self) -> Vec<MseDecodedFrame>`**：消费新到达的解码帧。每次调用应返回自上次调用以来累积的全部待处理帧。
- **`dimensions(&self) -> Option<(u32, u32)>`**：返回当前视频的分辨率，可能直到首帧解码后才返回 Some。
- **`state(&self) -> VideoSessionState`**：返回当前会话生命周期状态。

---

## 会话注册表

### `VideoFrameSessionId`
基于 `u64` 的 opaque 句柄，`Copy` + `Hash`，用于查找注册的会话。

### 全局状态
- **`VIDEO_FRAME_SESSION_IDS`**：`AtomicU64`，自增 ID 生成器，起始值为 1（0 保留作无效 ID）。
- **`VIDEO_FRAME_SESSIONS`**：`OnceLock<Mutex<HashMap<VideoFrameSessionId, Box<dyn VideoFrameSession>>>>`，全局注册表，懒初始化。

### `register_video_frame_session(session)`
向全局注册表中插入一个会话，返回新生成的不重复 `VideoFrameSessionId`。插入后会话所有权从调用方转移至注册表。

### `unregister_video_frame_session(id)`
从注册表中移除指定会话并返回其所有权。若 ID 不存在则返回 `None`。用于在绑定失败或放弃播放时清理注册。

### `take_registered_video_frame_session(id)`
`#[doc(hidden)]` 标记的内部方法，仅用于 widget/平台层在播放绑定时取走会话所有权。实际委托给 `unregister_video_frame_session`。

---

## 测试

`registry_roundtrip`：验证一个 StubSession 通过注册、取出、查询状态和解帧、再次取出的完整流程。断言 `take_registered_video_frame_session` 第二次调用返回 None。

`unregister_drops_pending_session`：验证先 unregister 后再 take 会返回 None，确认注册表项已被移除。
