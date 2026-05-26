# web_midi.rs — Web MIDI API 集成

**文件路径**: `/home/ubuntu/_github/makepad/platform/src/os/web/web_midi.rs` (162 行)
**核心作用**: Web 平台 MIDI 输入/输出管理，通过 `WebMidiAccess` 和 MPSC 通道实现 MIDI 数据传输，通过 WASM FFI 与 JavaScript Web MIDI API 桥接。

## 数据结构

### `OsMidiOutput` — MIDI 输出端

| 字段 | 类型 | 说明 |
|------|------|------|
| `sender` | `mpsc::Sender<(Option<MidiPortId>, MidiData)>` | 发送通道 |

- `send(port_id, d)` — 向 JS 侧发送 MIDI 数据，触发 UI 信号

### `OsMidiInput` — MIDI 输入端

| 字段 | 类型 | 说明 |
|------|------|------|
| `0` | `mpsc::Receiver<(MidiPortId, MidiData)>` | 接收通道 |

- `receive()` — 非阻塞读取 MIDI 输入数据

### `WebMidiPort` — Web MIDI 端口

| 字段 | 说明 |
|------|------|
| `uid` | JS 侧唯一标识符 |
| `desc` | Makepad `MidiPortDesc`（name, port_id, port_type） |

### `WebMidiAccess` — Web MIDI 访问状态（线程安全）

| 字段 | 类型 | 说明 |
|------|------|------|
| `output_receivers` | `Vec<mpsc::Receiver<...>>` | 输出通道接收端（用于消费 Rust→JS 的 MIDI 数据） |
| `input_senders` | `Vec<mpsc::Sender<...>>` | 输入通道发送端（用于分发 JS→Rust 的 MIDI 数据） |
| `change_signal` | `SignalToUI` | 端口列表变化信号 |
| `ports` | `Vec<WebMidiPort>` | 可用 MIDI 端口列表 |

## 核心方法

### 初始化与管理

- `WebMidiAccess::new(os, change_signal)` — 发送 `FromWasmQueryMidiPorts` 查询端口列表
- `create_midi_input()` → `MidiInput` — 创建 MPSC 通道，注册到 `input_senders`
- `create_midi_output()` → `MidiOutput` — 创建 MPSC 通道，注册到 `output_receivers`（作为 `OsMidiOutput`）

### 端口选择

- `use_midi_inputs(os, port_ids)` — 将选中的输入端口 UID 通过 `FromWasmUseMidiInputs` 发送到 JS
- `use_midi_outputs(os, ports)` — 当前空操作
- `midi_reset(os)` — 清空输入选择并重新查询端口列表

### 数据处理

- `to_wasm_midi_input_data(tw)` — 处理 JS 发来的 MIDI 输入数据：
  1. 按 `uid` 查找端口
  2. 将 32 位数据拆分为 3 字节 MIDI 消息
  3. 通过 `input_senders` 分发到对应输入通道
  4. 自动清理已断开的接收者

- `to_wasm_midi_port_list(tw)` — 处理 JS 发来的端口列表：
  1. 清空并重建端口列表
  2. 端口 ID 由 `LiveId::from_str(&uid)` 生成
  3. 触发 `change_signal`

- `send_midi_output_data(from_wasm)` — 消费输出通道中的待发送数据：
  1. 遍历 `output_receivers`
  2. 可选目标端口（`port_id.is_none()` 表示广播到所有端口，或精确匹配）
  3. 通过 `FromWasmSendMidiOutput` 发送到 JS

### CxOs 扩展

`CxOs::handle_web_midi_signals` — 在信号处理期间调用，消费并发送 MIDI 输出数据。

## 数据流

```
[硬件/DAW] → JS Web MIDI API → ToWasmMidiInputData → WebMidiAccess.to_wasm_midi_input_data()
    → mpsc::Sender → OsMidiInput.receive() → Makepad MIDI 系统

Makepad MIDI 系统 → OsMidiOutput.send() → mpsc::Receiver → WebMidiAccess.send_midi_output_data()
    → FromWasmSendMidiOutput → JS Web MIDI API → [硬件/DAW]
```
