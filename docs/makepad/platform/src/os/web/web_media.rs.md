# web_media.rs — Web 媒体管理（音频/MIDI/视频）

**文件路径**: `/home/ubuntu/_github/makepad/platform/src/os/web/web_media.rs` (113 行)
**核心作用**: Web 平台音频、MIDI 和视频媒体的集中管理，提供 `CxMediaApi` trait 实现，管理媒体资源的懒加载和信号分发。

## 关键数据结构

### `CxWebMedia`

```rust
pub struct CxWebMedia {
    pub(crate) web_audio: Option<Arc<Mutex<WebAudioAccess>>>,
    pub(crate) web_audio_change: SignalToUI,
    pub(crate) web_midi: Option<Arc<Mutex<WebMidiAccess>>>,
    pub(crate) web_midi_change: SignalToUI,
}
```

- `web_audio` / `web_midi` — 懒初始化的音频/MIDI 访问对象（线程安全）
- `web_audio_change` / `web_midi_change` — 设备列表变化的信号通知

## Cx 方法

### `handle_media_signals` — 媒体信号处理

在信号事件中调用：
1. 调用 `CxOs::handle_web_midi_signals()` 处理 MIDI 输出数据
2. 检查音频变化信号，若有变化通过 `get_updated_descs()` 获取设备列表并触发 `Event::AudioDevices`
3. 检查 MIDI 变化信号，若有变化触发 `Event::MidiPorts`

## CxOs 扩展

### 懒加载访问器

- `web_audio()` — 首次调用时创建 `WebAudioAccess` 并注册变化信号，之后返回缓存的 Arc
- `web_midi()` — 首次调用时创建 `WebMidiAccess` 并注册变化信号，之后返回缓存的 Arc

## `CxMediaApi` Trait 实现

| 方法 | 说明 |
|------|------|
| `midi_input()` | 通过 `WebMidiAccess::create_midi_input()` 创建 MIDI 输入 |
| `midi_output()` | 通过 `WebMidiAccess::create_midi_output()` 创建 MIDI 输出 |
| `midi_reset()` | 重置 MIDI（清空输入选择并重新查询端口） |
| `use_midi_inputs(ports)` | 选择使用的 MIDI 输入端口 |
| `use_midi_outputs(ports)` | 选择使用的 MIDI 输出端口 |
| `use_audio_inputs(devices)` | 选择使用的音频输入设备 |
| `use_audio_outputs(devices)` | 选择使用的音频输出设备 |
| `audio_output_box(index, f)` | 注册音频输出回调函数 |
| `audio_input_box(index, f)` | 注册音频输入回调函数 |
| `video_input_box(index, f)` | 空操作（Web 平台不支持视频输入回调） |
| `use_video_input(inputs)` | 空操作（Web 平台不支持视频输入选择） |
