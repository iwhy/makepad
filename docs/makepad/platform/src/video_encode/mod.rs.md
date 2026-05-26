# `video_encode/mod.rs` — 视频编码模块声明

## 概述
该文件是 `video_encode` 模块的入口，仅声明一个子模块：

- **`pub mod camera_video_encoder`**：提供 `VideoEncoder` 结构和工厂函数，封装 `MediaVideoEncoder` trait，管理视频编码会话的启动、帧推送、关键帧请求和停止。
