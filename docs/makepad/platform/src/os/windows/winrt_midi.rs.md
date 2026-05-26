# winrt_midi.rs — WinRT MIDI API 输入输出

**文件路径**: platform/src/os/windows/winrt_midi.rs (379行)
**核心用途**: 通过 Windows WinRT MIDI API (`Windows.Devices.Midi`) 实现 MIDI 设备枚举、设备热插拔监控、输入接收和输出发送。

## 结构体

### `OsMidiInput`
```rust
pub struct OsMidiInput(mpsc::Receiver<(MidiPortId, MidiData)>);
```

### `OsMidiOutput`
```rust
pub struct OsMidiOutput(pub(crate) Arc<Mutex<WinRTMidiAccess>>);
```

### `WinRTMidiPort`
```rust
pub struct WinRTMidiPort {
    winrt_id: String,       // WinRT 设备 ID
    desc: MidiPortDesc,     // 端口描述（名称、ID、类型）
}
```

### `WinRTMidiInput`
```rust
pub struct WinRTMidiInput {
    port_id: MidiPortId,
    event_token: i64,          // 用于取消事件订阅的 token
    midi_input: MidiInPort,    // WinRT MIDI 输入端口
}
```

### `WinRTMidiOutput`
```rust
pub struct WinRTMidiOutput {
    port_id: MidiPortId,
    midi_output: IMidiOutPort, // WinRT MIDI 输出端口
}
```

### `WinRTMidiAccess`
```rust
pub struct WinRTMidiAccess {
    input_senders: InputSenders,                   // 输入事件分发列表
    event_sender: mpsc::Sender<WinRTMidiEvent>,    // 控制通道发送端
    descs: Vec<MidiPortDesc>,                      // 缓存设备列表
}
```

## 枚举

### `WinRTMidiEvent`
```rust
enum WinRTMidiEvent {
    UpdateDevices,                        // 更新设备列表
    SendMidi(Option<MidiPortId>, MidiData),// 发送 MIDI 消息
    UseMidiInputs(Vec<MidiPortId>),       // 启用输入端口
    UseMidiOutputs(Vec<MidiPortId>),      // 启用输出端口
}
```

## 工作线程架构

所有 WinRT MIDI 操作在独立的监控线程中执行：

```
┌── 主线程 ──┐                    ┌── 监控线程 ──────────────────────┐
│            │  event_sender      │                                  │
│ WinRTMidi  │ ──────────────────→│  while let Ok(msg) = receiver    │
│ Access     │                    │    match msg {                   │
│            │                    │      UpdateDevices → get_ports   │
│            │                    │      UseMidiInputs → open/close  │
│            │                    │      UseMidiOutputs → open/close │
│            │                    │      SendMidi → write & send     │
│            │                    │    }                             │
└────────────┘                    └──────────────────────────────────┘
```

## 关键函数

### `WinRTMidiAccess::new`
1. 创建 mpsc 通道（`watch_sender`/`watch_receiver`）
2. 启动监控线程：
   - 发送初始 `UpdateDevices` 事件
   - 创建输入和输出 `DeviceWatcher`
   - 绑定 `Added`、`Removed`、`Updated`、`EnumerationCompleted` 事件 → 发送 `UpdateDevices`
   - 启动 watcher
   - 循环处理 `WinRTMidiEvent`

### 设备枚举

`get_ports_list` (async):
1. `MidiInPort::GetDeviceSelector()` 查询输入设备
2. `DeviceInformation::FindAllAsyncAqsFilter()` 枚举
3. 对每个设备：获取 ID、名称
4. 同样方式枚举输出设备
5. 返回 `Vec<WinRTMidiPort>`

### 端口管理

| 方法 | 描述 |
|------|------|
| `use_midi_inputs(ports)` | 发送 `UseMidiInputs` 控制消息 |
| `use_midi_outputs(ports)` | 发送 `UseMidiOutputs` 控制消息 |
| `midi_reset` | 关闭所有输入/输出并重新枚举设备 |
| `create_midi_input` | 创建新的 `MidiInput`（mpsc 接收器），注册到 `input_senders` |

### 输入处理

在监控线程中：
1. 对每个需要启用的输入端口：
   - 调用 `MidiInPort::FromIdAsync` 异步打开
   - 注册 `MessageReceived` 事件处理器
2. 事件处理器：
   - `msg.Message()` → `RawData()` → `DataReader::FromBuffer`
   - 读取 3 字节 MIDI 数据
   - 通过所有注册的 `input_senders` 分发
   - 调用 `SignalToUI::set_ui_signal()` 唤醒主线程
3. 对不再需要的端口：
   - `RemoveMessageReceived(event_token)` → `Close()`

### 输出处理

在监控线程中：
1. 对每个需要启用的输出端口：
   - 调用 `MidiOutPort::FromIdAsync` 异步打开
2. `SendMidi` 事件：
   - `DataWriter::new()` → `WriteBytes` → `DetachBuffer`
   - 对所有匹配端口调用 `midi_output.SendBuffer()`

## 平台集成

- 比 `win32_midi.rs` 更可靠：支持热插拔、异步 API、更稳定的设备枚举
- 通过 `CxWindowsMedia::winrt_midi()` 延迟初始化
- `CxMediaApi` 桥接（见 `windows_media.rs`）
- 使用 `makepad_futures_legacy::executor::block_on` 在同步上下文中执行异步调用
