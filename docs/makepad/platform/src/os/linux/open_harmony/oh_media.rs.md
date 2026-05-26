# oh_media.rs

One-liner (EN): Stub implementation of Media (MIDI, audio, video) API for the OpenHarmony platform.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/open_harmony/oh_media.rs` (66 行)
- **核心作用**: 为 OpenHarmony 平台提供 `CxMediaApi` trait 的桩（stub）实现。所有 MIDI、音频、视频接口均为空操作，不执行任何实际功能。

## 类型/结构体

| 类型 | 说明 |
|------|------|
| `OsMidiOutput` | MIDI 输出桩，`send` 方法为空 |
| `OsMidiInput` | MIDI 输入桩，`receive` 恒返回 `None` |
| `CxOpenHarmonyMedia` | 媒体状态容器（`#[derive(Default)]`），无字段 |

## key 方法/函数

### `impl Cx`

| 方法 | 用途 |
|------|------|
| `handle_media_signals()` | 处理媒体信号回调（空实现） |
| `reinitialise_media()` | 重新初始化媒体子系统（空实现） |

### `impl CxMediaApi for Cx`

| 方法 | 行为 |
|------|------|
| `midi_input() / midi_output()` | 返回包装了 `OsMidiInput`/`OsMidiOutput` 的相应结构 |
| `midi_reset()` | 空操作 |
| `use_midi_inputs() / use_midi_outputs()` | 空操作（忽略端口列表） |
| `use_audio_inputs() / use_audio_outputs()` | 空操作（忽略设备列表） |
| `audio_output_box() / audio_input_box()` | 空操作（忽略索引和回调） |
| `video_input_box()` | 空操作（忽略索引和回调） |
| `use_video_input()` | 空操作（忽略输入/格式列表） |

## 平台集成

此为 OpenHarmony 的暂时实现。所有媒体功能均为空操作，意味着在当前版本中：
- MIDI 输入/输出不可用
- 音频捕获和播放不可用
- 视频捕获不可用

未来版本可在此文件中添加基于 OpenHarmony 原生多媒体框架（AVPlayer、AVRecorder 等）的实际实现。
