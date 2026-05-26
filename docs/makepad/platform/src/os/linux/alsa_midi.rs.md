# alsa_midi.rs — ALSA MIDI 端口管理与事件处理

**文件路径**: `platform/src/os/linux/alsa_midi.rs` (510 行)
**核心功能**: 通过 ALSA Sequencer API 实现 MIDI 输入/输出端口的枚举、订阅和事件收发。

## 主要类型

### `AlsaMidiAccess`
MIDI 访问管理器：
- `input_senders` — MIDI 输入事件的分发通道列表
- `ports` — 当前已知的 MIDI 端口列表
- `client` — ALSA Sequencer 客户端（输入+输出）

### `OsMidiOutput` / `OsMidiInput`
对外暴露的 MIDI 输出/输入包装类型：
- `OsMidiOutput` — 通过 `send` 发送 MIDI 数据
- `OsMidiInput` — 通过 `receive` 非阻塞接收 MIDI 数据

### `AlsaClient`
ALSA Sequencer 客户端封装：
- `in_client` / `out_client` — 独立的输入和输出 sequencer 句柄
- `midi_send` — MIDI 事件编码器

### `AlsaMidiPort`
单个 MIDI 端口描述：
- `client_id` / `port_id` — ALSA 客户端和端口标识
- `subscribed` — 是否已订阅
- `desc` — MIDI 端口描述（名称、类型等）

## 关键方法

### `AlsaMidiAccess::new(change_signal)`
创建 MIDI 访问实例：
- 创建 ALSA Sequencer 输入和输出客户端
- 创建输入端口并订阅系统 announce 端口以监听设备变化
- 启动后台事件循环线程，持续读取 `snd_seq_event_input`
- 将 ALSA MIDI 事件转换为框架的 `MidiData` 类型（NoteOn/Off、ControlChange、PitchBend 等）
- 通过 `input_senders` 分发到各接收者

### `AlsaMidiAccess::get_updated_descs()`
断开所有已订阅端口，重新枚举系统中的 MIDI 端口：
- 遍历所有 ALSA Sequencer 客户端
- 检查每个端口的 capabilities（读写能力）
- 过滤出支持 MIDI 输入/输出的端口

### `AlsaMidiAccess::send_midi(port_id, data)`
将 `MidiData` 编码为 `snd_seq_event_t` 并通过 `snd_seq_event_output_direct` 发送到指定（或所有）输出端口。

### `AlsaMidiAccess::use_midi_inputs/outputs(ports)`
启用/禁用指定 MIDI 端口的订阅。遍历端口列表，为目标端口创建 `snd_seq_port_subscribe_t` 连接。

## 事件处理

后台事件循环处理以下 ALSA MIDI 事件类型：
- `SND_SEQ_EVENT_NOTEON/NOTEOFF` → `MidiNote`
- `SND_SEQ_EVENT_KEYPRESS` → `MidiAftertouch`
- `SND_SEQ_EVENT_CONTROLLER` → `MidiControlChange`
- `SND_SEQ_EVENT_PGMCHANGE` → `MidiProgramChange`
- `SND_SEQ_EVENT_CHANPRESS` → `MidiChannelAftertouch`
- `SND_SEQ_EVENT_PITCHBEND` → `MidiPitchBend`
- 端口变化事件 → 触发 `change_signal`

## 实现细节

- 输入和输出使用独立的 ALSA Sequencer 客户端
- 使用 `mpsc::channel` 实现线程安全的 MIDI 事件分发
- 使用 `alsa_error!` 宏（复用自 `alsa_audio.rs`）处理 ALSA 错误
