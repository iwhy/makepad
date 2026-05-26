# `midi.rs` — MIDI 端口管理与事件编解码

## 概述
该文件实现了 Makepad MIDI 子系统的核心类型，涵盖 MIDI 端口枚举、MIDI 数据编解码、以及各类 MIDI 消息（音符、控制变化、弯音等）的结构化表示。

---

## 端口枚举与事件

### `MidiPortsEvent`
当系统 MIDI 端口列表发生变化时触发的事件，内部维护 `Vec<MidiPortDesc>`。
- **`all_inputs()`**：筛选所有类型为 Input 的端口 ID 列表。
- **`all_outputs()`**：筛选所有类型为 Output 的端口 ID 列表。

### `MidiPortType`
端口方向枚举：`Input`（接收 MIDI 消息）、`Output`（发送 MIDI 消息）。提供 `is_input()` / `is_output()` 快速判断。

### `MidiPortId`
基于 `LiveId` 的端口唯一标识符，支持 `FromLiveId` 宏生成和 `Hash`/`Eq` 比较。

### `MidiPortDesc`
端口描述：名称、端口 ID、端口类型。

---

## MIDI I/O

### `MidiInput`
封装 `OsMidiInput` 的 MIDI 输入包装器，默认构造为空（`Default` trait）。
- **`receive()`**：从底层系统 MIDI 输入接收一条消息，返回 `Option<(MidiPortId, MidiData)>`，包含来源端口 ID 和 3 字节 MIDI 数据。

### `MidiOutput`
封装 `OsMidiOutput` 的 MIDI 输出包装器，默认构造为空。
- **`send(port, data)`**：向指定端口（可选，`None` 表示默认端口）发送 3 字节 MIDI 消息。

---

## MIDI 数据表示

### `MidiData`
3 字节的原始 MIDI 消息，`data: [u8; 3]`。
- **`From<u32>`**：将 u32 的高 3 字节解包为 MIDI 数据，格式为 `[byte2, byte1, byte0]`。
- **`status()`**：提取状态字节的高 4 位（`data[0] >> 4`），用于区分消息类型。
- **`channel()`**：提取状态字节的低 4 位（`data[0] & 0xf`），表示 MIDI 通道号。

### `MidiData::decode()` — MIDI 消息解码

根据状态码将原始 3 字节解码为结构化 `MidiEvent`：

- **`0x8`（Note Off）或 `0x9`（Note On，力度 0 视为 Note Off）** → `MidiEvent::Note`，包含开关标志、通道、音符号、力度。
- **`0xA`（Polyphonic Aftertouch）** → `MidiEvent::Aftertouch`，通道、音符号、压力值。
- **`0xB`（Control Change）** → `MidiEvent::ControlChange`，通道、控制器编号、值。
- **`0xC`（Program Change）** → `MidiEvent::ProgramChange`，通道、音色库高位/低位。
- **`0xD`（Channel Aftertouch）** → `MidiEvent::ChannelAftertouch`，通道、14 位压力值（由 byte1<<7 | byte2 拼合）。
- **`0xE`（Pitch Bend）** → `MidiEvent::PitchBend`，通道、14 位弯音值。
- **`0xF`（System）** → `MidiEvent::System`，通道、数据高位/低位。
- **其他** → `MidiEvent::Unknown`，保留原始数据。

---

## MIDI 消息类型

每个消息类型都实现了 `Into<MidiData>` 以编码为 3 字节格式：

### `MidiNote`
音符开/关：
- **`is_on`**：`true` 为 Note On，`false` 为 Note Off。
- **`channel`**：通道号（0-15）。
- **`note_number`**：音符号（0-127）。
- **`velocity`**：力度（0-127）。
- 编码：状态字节 `(0x9|0x8)<<4 | channel`，字节 1 为音符号，字节 2 为力度。

### `MidiAftertouch`
复音触后（Polyphonic Key Pressure）：
- 编码：状态字节 `0xA0 | channel`。

### `MidiControlChange`
控制器变化：
- **`param`**：控制器编号。
- **`value`**：控制器值。
- 编码：状态字节 `0xB0 | channel`。

### `MidiProgramChange`
音色变化：
- 编码：状态字节 `0xC0 | channel`，字节 1 和 2 为音色数据。

### `MidiChannelAftertouch`
通道触后（Channel Pressure）：
- **`value`**：14 位压力值。
- 编码：状态字节 `0xD0 | channel`，高位/低位分别存入字节 1/2。

### `MidiPitchBend`
弯音轮：
- **`bend`**：14 位弯音值。
- 编码：状态字节 `0xE0 | channel`，高位/低位分别存入字节 1/2。

### `MidiSystem`
系统专用消息：
- 编码：状态字节 `0xF0 | channel`。

---

## `MidiEvent`
所有 MIDI 事件的联合枚举：
- `Note(MidiNote)` / `Aftertouch(MidiAftertouch)` / `ControlChange(MidiControlChange)` / `ProgramChange(MidiProgramChange)` / `PitchBend(MidiPitchBend)` / `ChannelAftertouch(MidiChannelAftertouch)` / `System(MidiSystem)` / `Unknown(MidiData)`
- **`on_note()`**：便捷方法，若事件为 Note 则返回 `Some(MidiNote)`，否则返回 None。
