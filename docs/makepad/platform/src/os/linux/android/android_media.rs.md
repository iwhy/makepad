# Android 媒体子系统聚合

## 概述

`android_media.rs` 聚合了 Android 平台的音频、MIDI 和摄像头媒体子系统。它管理子系统的懒初始化、信号分发，并实现 `CxMediaApi` trait 供 Makepad 核心调用。

## 核心类型

### `CxAndroidMedia`
`Default` 实现。包含：
- `android_audio_change` / `android_midi_change` / `android_camera_change` — 信号量
- `android_audio` / `android_midi` / `android_camera` — 各子系统的 `Arc<Mutex<>>` 句柄（`Option`，懒初始化）

## 信号处理

### `Cx::handle_media_signals()`
在主事件循环中调用，检查三个信号量：
1. **音频变更** → 调用 `get_updated_descs()`，派发 `Event::AudioDevices`
2. **MIDI 变更** → 调用 `get_updated_descs()`，派发 `Event::MidiPorts`
3. **摄像头变更** → 如果 `android_camera` 尚未初始化，先初始化再获取描述，派发 `Event::VideoInputs`

### `Cx::reinitialise_media()`
设置所有三个信号量，触发重新枚举。

## 懒初始化

- `android_audio()` / `android_midi()` / `android_camera()` 方法在首次调用时创建对应子系统
- 创建后缓存为 `Arc<Mutex<>>`，后续调用返回同一实例

## CxMediaApi 实现

| 方法 | 委托给 |
|------|--------|
| `midi_input` / `midi_output` | `AndroidMidiAccess::create_midi_input` |
| `midi_reset` | 空操作 |
| `use_midi_inputs` / `use_midi_outputs` | `AndroidMidiAccess` |
| `use_audio_inputs` / `use_audio_outputs` | `AndroidAudioAccess` |
| `audio_output_box` / `audio_input_box` | 设置回调到 `AndroidAudioAccess` |
| `video_input_box` / `camera_frame_input_box` | 设置回调到 `AndroidCameraAccess` |
| `video_encoder_output_box` | 配置视频编码器 |
| `video_encoder_push_frame` | 推送视频帧到编码器 |
| `video_encoder_capture_texture_frame` | 捕获纹理帧 |
| `video_encoder_request_keyframe` | 请求关键帧 |
| `video_capabilities` | 查询 H.264 编码/解码能力 |
| `use_video_input` | 激活摄像头输入 |

## 视频能力查询

`video_capabilities()` 通过 JNI 调用 `to_java_query_h264_codec_support()` 获取硬件/软件编解码能力，填充 `VideoCodecSupport`：
- H.264：编码格式 `AnnexB`，解码格式 `AnnexB` + `Avcc`
- H.265：始终标记为不支持
- 与 `crate::media_video_capabilities()` 合并

## 实现说明

- 此文件是 Android 媒体子系统的统一入口点，不实现具体设备操作。
- `handle_media_signals` 在 `android.rs` 的事件循环中被调用。
- `video_capabilities` 使用 `unsafe` 的 JNI 调用探测硬件编解码能力。
