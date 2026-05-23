# window_voice_input.rs — 语音输入模块（条件编译）

## 文件概述

`window_voice_input` 是 Makepad 的语音输入模块，仅在启用 `feature = "voice"` 时编译。提供完整的语音采集、VAD（Voice Activity Detection）、降噪和实时转写功能。

---

## 条件编译

```rust
// lib.rs
#[cfg(feature = "voice")]
mod window_voice_input;
```

该模块依赖 Whisper.cpp 或其他语音识别库，编译时需要开启 `voice` feature：

```bash
cargo build --features voice
```

---

## WindowVoiceInput 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct WindowVoiceInput {
    #[deref] view: View,
    #[live] animator: Animator,
    #[live] mic_button: Button,          // 麦克风按钮
    #[live] waveform: Option<WidgetRef>, // 波形显示
    #[live] status_label: Label,         // 状态提示
    #[rust] audio_stream: Option<AudioStream>,  // 音频流
    #[rust] is_listening: bool,
    #[rust] is_processing: bool,
    #[rust] transcription: String,       // 当前转写结果
}
```

---

## 语音处理管线

```
麦克风输入 → VAD → 降噪 → 缓存 → 转写
    ↑                            ↓
  按钮触发                  文本输出到 UI
```

### 1. 音频采集

从系统麦克风采集音频流：

```rust
fn start_listening(&mut self, cx) {
    // 1. 请求麦克风权限
    // 2. 创建音频流
    // 3. 开始采集（16kHz 单声道 PCM）
    self.audio_stream = Some(AudioStream::start(cx));
    self.is_listening = true;
}
```

### 2. VAD（Voice Activity Detection）

检测说话的开始和结束：

- **静音阈值**：当音频能量低于阈值时判定为静音。
- **说话起始**：连续 N 帧高于阈值的音频。
- **说话结束**：连续 M 帧低于阈值的音频（M = hangover 帧数）。
- 检测到说话结束后，缓存片段发送到转写引擎。

### 3. 降噪

使用基本频谱减法或深度学习降噪模型：

- 估计背景噪声谱（语音间歇期）。
- 从当前信号中减去噪声成分。
- 输出降噪后的音频片段。

### 4. 转写

将音频片段发送到语音识别引擎（Whisper.cpp）：

```rust
fn transcribe_audio(&mut self, audio: Vec<f32>) {
    // 1. 缓存音频片段
    // 2. 通过通道发送到异步线程处理
    // 3. 异步线程调用 Whisper.cpp API
    // 4. 主线程接收转写结果
}
```

### 5. 结果输出

转写完成的文本发送到关联的 `TextInput` 或 `TextArea`：

```rust
fn on_transcription_result(&mut self, cx, text: String) {
    self.transcription = text.clone();
    if let Some(input) = self.target_text_input {
        input.set_text(cx, &text);
    }
}
```

---

## UI 交互

### 麦克风按钮

- 点击启动/停止录音。
- 录音中显示红色指示灯。
- 按钮状态绑定 Animator，实现 pulse 动画效果。

### 波形显示

`waveform` 可选组件，在录音时实时显示音频波形：

```
╭───╮     ╭───╮     ╭─╮
│   ╰─╮ ╭─╯   ╰─╮ ╭╯ ╰╮
│     ╰─╯       ╰─╯   │
```

波形振幅映射音频采样值的 RMS 级别。

### 状态提示

`status_label` 显示当前状态：

| 状态 | 文本 | 颜色 |
|------|------|------|
| 空闲 | "点击开始录音" | 默认 |
| 录音中 | "正在录音..." | 红色 |
| 处理中 | "正在识别..." | 橙色 |
| 完成 | "识别完成" | 绿色 |
| 错误 | "录音失败，请检查麦克风权限" | 红色 |

---

## 线程模型

```rust
struct TranscribeTask {
    sender: UiHandle<String>,  // 跨线程发送转写结果
}

impl TranscribeTask {
    fn run(audio: AudioBuffer) {
        // 在后台线程执行
        let text = whisper::transcribe(audio);  // 耗时操作
        self.sender.send(text);  // 安全发送回 UI 线程
    }
}
```

- 音频采集：在主线程（low-latency 回调）。
- 转写处理：异步线程（CPU 密集型）。
- 结果更新：通过 `UiHandle` 发送回主线程。
- 使用 `ScriptAsync` 管理异步生命周期。

---

## 依赖关系

```
feature = "voice"
  ├── whisper.cpp 或 bindings
  ├── cpal（音频采集）
  └── rubato（可选，重采样）
```

模块使用 `cfg` gate 隔离所有平台相关依赖，非 `voice` 编译时完全不包含相关代码。
