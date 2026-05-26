# core_midi.rs — CoreMIDI 输入

**文件路径:** `platform/src/os/apple/core_midi.rs`

**核心目的:** 使用 Apple CoreMIDI 框架接收 MIDI 输入。支持枚举 MIDI 源、建立 MIDI 输入连接、接收 Note On/Off、控制变更等 MIDI 消息。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `CoreMidiInput` | MIDI 输入管理器，封装 MIDI 客户端和输入端口 |
| `MidiMessage` | 解析后的 MIDI 消息结构体，包含状态字节、数据字节和时间戳 |

**关键方法:**
- `CoreMidiInput::new(client_name, cb)` — 创建 MIDI 输入客户端，指定接收回调
- `CoreMidiInput::get_sources()` — 枚举所有可用的 MIDI 输入源
- `CoreMidiInput::connect_source(source_id)` — 连接到指定的 MIDI 输入源
- `CoreMidiInput::disconnect_source(source_id)` — 断开 MIDI 输入源
- `CoreMidiInput::dispose()` — 销毁 MIDI 客户端和端口
- `MidiMessage::from_raw_packet(packet)` — 从 `MIDIPacket` 解析 MIDI 消息

**`MidiMessage` 类型:**
- `NoteOn { channel, note, velocity }`
- `NoteOff { channel, note, velocity }`
- `ControlChange { channel, controller, value }`
- `ProgramChange { channel, program }`
- `PitchBend { channel, value }`
- `Aftertouch { channel, note, pressure }`
- `ChannelPressure { channel, pressure }`
- `SysEx { data }`
- `Unknown { status, data }`

**实现细节:**
- 使用 `MIDIClientCreate` 创建 MIDI 客户端
- 使用 `MIDIInputPortCreate` 创建输入端口
- 通过 `MIDIPortConnectSource` 连接到 MIDI 源
- MIDI 回调使用 `block!` 宏创建 Objective-C block
- `MIDIPacket` 解析遵循标准 MIDI 协议
- 支持运行状态（Running Status）消息
- 实时时钟和系统消息也传递通过

**平台集成:** macOS 和 iOS，使用 CoreMIDI 框架
