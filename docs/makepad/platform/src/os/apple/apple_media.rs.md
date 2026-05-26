# apple_media.rs — Apple 媒体元数据

**文件路径:** `platform/src/os/apple/apple_media.rs`

**核心目的:** 通过 `iTunesMusicAPI`/`MusicAPI` 与 Apple 的 Music.app 通信，获取当前播放曲目的元数据。适用于集成 Now Playing 信息显示。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `AppleMediaTrack` | 曲目信息结构体，包含标题、艺术家、专辑、封面图片数据、播放状态和时间信息 |

**关键函数:**
- `get_current_media_track(wait_for_title_ms)` — 获取当前在 Music.app 中播放的曲目信息：
  - 通过分布式通知监听曲目变化
  - 使用 AppleScript 或 ScriptingBridge 查询 Music.app
  - 返回 `Option<AppleMediaTrack>`

**`AppleMediaTrack` 字段:**
- `title: String` — 曲目标题
- `artist: String` — 艺术家
- `album: String` — 专辑名
- `artwork_data: Option<Vec<u8>>` — 封面图片的二进制数据（通常是 JPEG/PNG）
- `state: MediaPlaybackState` — 播放状态（Playing/Paused/Stopped）
- `current_time: f64` — 当前播放位置（秒）
- `duration: f64` — 曲目总时长（秒）

**实现细节:**
- macOS 上通过 `NSDistributedNotificationCenter` 监听 `com.apple.iTunes.playerInfo` 通知
- 从通知的 `userInfo` 字典提取 TrackID、Artist、Name、Album、State、PlayerPosition、Duration
- 封面图片通过 `iTunesArtworkURL` 条目中的持久 ID 从 `iTunes Music Library.xml` 定位
- iOS 上使用 `MPNowPlayingInfoCenter` 获取信息
- 超时机制：`wait_for_title_ms` 参数控制等待元数据变化的最长时间

**平台集成:** macOS 专用（iTunes/Music.app 通知）；iOS 使用 `MPNowPlayingInfoCenter`
