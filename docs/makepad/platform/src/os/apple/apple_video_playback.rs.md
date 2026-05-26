# apple_video_playback.rs — 视频播放流水线

**文件路径:** `platform/src/os/apple/apple_video_playback.rs`

**核心目的:** 使用 AVFoundation 实现高性能视频播放流水线。提供 `VideoPlayer` 结构体，封装 AVPlayer、AVPlayerItem 和视频输出到 Metal 纹理的完整流程。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `VideoPlayer` | 视频播放器主结构体 |
| `VideoPlayerState` | 播放状态枚举（`Idle`, `Loading`, `Ready`, `Playing`, `Paused`, `Ended`, `Failed`） |
| `VideoFrame` | 解码后的视频帧，包含 Metal 纹理引用和时间戳 |
| `VideoTrackInfo` | 视频轨道信息（尺寸、编码、帧率） |

**关键方法:**
- `VideoPlayer::new(url)` — 创建视频播放器实例：
  - 创建 `AVURLAsset`，加载视频资源
  - 创建 `AVPlayerItem`，关联到 asset
  - 创建 `AVPlayer`，配置播放参数
- `VideoPlayer::play()` — 开始/恢复播放
- `VideoPlayer::pause()` — 暂停播放
- `VideoPlayer::seek_to(time)` — 跳转到指定时间位置
- `VideoPlayer::set_rate(rate)` — 设置播放速率（慢放/快进）
- `VideoPlayer::set_volume(v)` / `set_muted(m)` — 音量控制
- `VideoPlayer::current_time()` — 获取当前播放时间
- `VideoPlayer::duration()` — 获取视频总时长
- `VideoPlayer::get_current_frame()` — 获取当前解码帧（Metal 纹理格式）

**视频输出流水线:**
1. AVPlayer 解码视频帧到 `CVPixelBuffer`（NV12 格式）
2. `AVPlayerItemVideoOutput` 从显示链接回调中拉取新帧
3. `CVMetalTextureCache` 将 `CVPixelBuffer` 转换为 Metal 纹理
4. YUV 平面纹理传递到 `DrawYuvMetal` 进行颜色转换
5. RGB 渲染到目标 Metal drawable

**状态管理:**
- 通过 KVO (Key-Value Observing) 监控 `AVPlayerItem` 的 `status`、`playbackBufferEmpty`、`playbackLikelyToKeepUp` 等属性
- 通过 `AVPlayerItemDidPlayToEndTimeNotification` 检测播放结束
- 时间观察者通过 `addPeriodicTimeObserverForInterval` 实现
- 错误状态通过 `AVPlayerItemFailedToPlayToEndTimeNotification` 检测

**实现细节:**
- 使用 `AVPlayerItemVideoOutput` 搭配 `CADisplayLink` 实现帧同步
- `CVPixelBuffer` 到 Metal 纹理的零拷贝转换（通过 `CVMetalTextureCacheCreateTextureFromImage`）
- 支持 HDR 视频（`kCVPixelFormatType_420YpCbCr10BiPlanarVideoRange` 等）
- 音频播放自动路由到系统输出
- 支持本地文件和远程 URL 播放
- 背景播放通过 `AVAudioSession` 配置

**平台集成:** macOS 和 iOS/tvOS，使用 AVFoundation、CoreVideo、Metal 框架
