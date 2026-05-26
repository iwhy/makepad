# Android 视频播放配置

## 概述

`android_video_playback.rs` 定义了 Android 视频播放的配置结构和格式支持检测。它决定了 Makepad 在 Android 上使用原生（Java MediaPlayer）还是软件（Rust ffmpeg）视频播放路径。

## 核心类型

### `AndroidVideoConfig`
视频播放配置（`Clone`）：
- `video_id` — 唯一视频标识符
- `source` — 视频源（内存/网络/文件系统/摄像头/会话）
- `texture_id` — 外部纹理 ID
- `tex_y_id` / `tex_u_id` / `tex_v_id` — YUV 平面纹理 ID
- `autoplay` — 是否自动播放
- `should_loop` — 是否循环播放

## 函数

### `force_software_video() -> bool`
检查环境变量 `MAKEPAD_FORCE_SOFTWARE_VIDEO` 是否设置。设置后强制使用软件视频解码器。

### `force_native_video() -> bool`
检查环境变量 `MAKEPAD_FORCE_NATIVE_VIDEO` 是否设置。设置后强制使用原生 Java MediaPlayer。

### `can_play_type(mime: &str) -> &'static str`
Android 平台 `canPlayType` 查表实现。返回 `"probably"`、`"maybe"` 或 `""`：

| MIME 类型 | 结果 |
|-----------|------|
| `video/mp4`、`video/x-m4v` | `"probably"` |
| `audio/mp4`、`audio/x-m4a`、`audio/mpeg`、`audio/wav` | `"probably"` |
| `video/webm`、`audio/webm`、`video/ogg`、`audio/ogg` | `"maybe"` |
| 其他 `video/` 或 `audio/` 前缀 | `"maybe"` |
| 其他 | `""` |

## 实现说明

- 此文件不含视频播放的实际逻辑，只做配置和格式检测。
- 原生路径通过 `android_jni.rs` 的 `to_java_prepare_video_playback` 等函数与 Java MediaPlayer 交互。
- 软件路径在 `video_decode::software_video` 模块中实现。
- 环境变量设置后影响全局行为，建议仅在调试/测试场景使用。
