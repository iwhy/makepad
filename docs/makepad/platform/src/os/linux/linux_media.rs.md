# linux_media.rs — Linux 媒体子系统编排

**文件路径**: `platform/src/os/linux/linux_media.rs` (252 行)
**核心功能**: 整合 ALSA 音频、ALSA MIDI 和 V4L2 摄像头三大媒体子系统，实现 `CxMediaApi` trait，协调设备变更信号和回调注册。

## 主要类型

### `CxLinuxMedia`
Linux 媒体子系统状态的容器：
- `pulse_audio` / `alsa_audio` — 可选音频后端
- `alsa_midi` — MIDI 访问
- `v4l2_camera` — 摄像头访问
- `audio_change` / `alsa_midi_change` / `v4l2_change` — 设备变更信号

## 关键方法

### `Cx::handle_media_signals()`
事件循环中调用的主媒体信号处理器：
- 检测音频设备变更（按优先级 ALSA > PulseAudio）
- 检测 MIDI 端口变更
- 延迟初始化 V4L2 摄像头子系统
- 通过 `call_event_handler` 发送 `AudioDevicesEvent`、`MidiPortsEvent`、`VideoInputsEvent`

### `CxLinuxMedia` 惰性初始化方法
- `alsa_audio()` / `pulse_audio()` — 首次访问时创建实例
- `alsa_midi()` — 首次访问时创建 MIDI 访问
- `v4l2_camera()` — 首次访问时创建摄像头访问

## `CxMediaApi` 实现

### 音频
- `use_audio_inputs/outputs` — 同时管理 ALSA 和 PulseAudio 后端
- `audio_output_box/input_box` — 注册音频回调
- PulseAudio 可通过环境变量 `MAKEPAD_DISABLE_PULSE_AUDIO` 禁用

### MIDI
- `midi_input()` / `midi_output()` — 创建 MIDI 输入/输出
- `use_midi_inputs/outputs` — 订阅/取消订阅 MIDI 端口

### 视频
- `video_input_box` / `camera_frame_input_box` — 注册视频输入回调
- `video_encoder_output_box` — 注册视频编码回调（仅支持摄像头源）
- `use_video_input` — 选择视频输入格式

### 视频能力
- `video_capabilities()` — 返回合并后的视频能力信息
