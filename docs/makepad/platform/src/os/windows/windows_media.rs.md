# windows_media.rs — Windows 媒体子系统桥接

**文件路径**: platform/src/os/windows/windows_media.rs (208行)
**核心用途**: 聚合 WASAPI（音频）、Media Foundation（视频输入）和 WinRT MIDI 子系统，实现 `CxMediaApi` trait 的媒体回调桥接。

## 结构体

### `CxWindowsMedia`
```rust
pub struct CxWindowsMedia {
    pub(crate) winrt_midi: Option<Arc<Mutex<WinRTMidiAccess>>>,
    pub(crate) wasapi: Option<Arc<Mutex<WasapiAccess>>>,
    pub(crate) media_foundation: Option<Arc<Mutex<MediaFoundationAccess>>>,
    pub(crate) wasapi_change: SignalToUI,
    pub(crate) media_foundation_change: SignalToUI,
    pub(crate) winrt_midi_change: SignalToUI,
}
```

## 关键方法

### 延迟初始化

| 方法 | 子模块 | 初始化时机 |
|------|--------|-----------|
| `winrt_midi()` | WinRT MIDI | 首次访问时创建 |
| `wasapi()` | WASAPI 音频 | 首次访问时创建 |
| `media_foundation()` | Media Foundation 摄像头 | 首次访问时创建 |

### `Cx::handle_media_signals`
在主事件循环中检查三个变更信号：
- `winrt_midi_change`: 触发 `Event::MidiPorts(MidiPortsEvent)`
- `wasapi_change`: 触发 `Event::AudioDevices(AudioDevicesEvent)`
- `media_foundation_change`: 触发 `Event::VideoInputs(VideoInputsEvent)`

### CxMediaApi 实现

| 方法 | 代理到 |
|------|--------|
| `midi_input()` | `winrt_midi().create_midi_input()` |
| `midi_output()` | `OsMidiOutput(winrt_midi())` |
| `use_midi_inputs` | `winrt_midi().use_midi_inputs()` |
| `use_midi_outputs` | `winrt_midi().use_midi_outputs()` |
| `midi_reset` | `winrt_midi().midi_reset()` |
| `use_audio_inputs` | `wasapi().use_audio_inputs()` |
| `use_audio_outputs` | `wasapi().use_audio_outputs()` |
| `audio_output_box` | 设置 `wasapi().audio_output_cb[index]` |
| `audio_input_box` | 设置 `wasapi().audio_input_cb[index]` |
| `video_input_box` | 设置 `media_foundation().video_input_cb[index]` |
| `camera_frame_input_box` | 设置 `media_foundation().camera_frame_input_cb[index]` |
| `use_video_input` | `media_foundation().use_video_input()` |
| `video_encoder_output_box` | 配置 `VideoEncoder` 并绑定输出回调（仅支持 `VideoCodec::Av1` + `VideoEncodeSource::Camera`） |
| `video_capabilities` | 返回合并的 `VideoCapabilities` |

### 视频编码支持

`video_encoder_output_box` 仅支持：
- 编码器: `VideoCodec::Av1`
- 源类型: `VideoEncodeSource::Camera`（通过 Media Foundation 捕获 + 软件编码器）
- `Texture` 和 `CpuFrames` 源返回 `Err(VideoEncodeError::UnsupportedSource)`
