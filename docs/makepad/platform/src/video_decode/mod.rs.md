# `video_decode/mod.rs` — 软件视频解码模块声明

## 概述
该文件是 `video_decode` 模块的入口文件，声明并导出两个子模块：

- **`pub mod software_video`**：提供 `PlaybackSessionHandle`，即媒体回放会话的软件解码适配层，封装 `MediaPlaybackSession` trait，管理`视频播放的启动、帧轮询、播放控制等。
- **`pub mod yuv`**：提供 `YuvPlaneData`、`YuvLayout`、`YuvColorMatrix` 等 YUV 数据类型，是解码后帧数据的核心表示。
