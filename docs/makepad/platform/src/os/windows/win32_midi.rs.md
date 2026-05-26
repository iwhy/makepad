# win32_midi.rs — Win32 MIDI API 输入输出

**文件路径**: platform/src/os/windows/win32_midi.rs (315行)
**核心用途**: 通过 Win32 MIDI API (`midiIn*` / `midiOut*`) 实现 MIDI 设备枚举、输入接收和输出发送。

## 结构体

### `OsMidiOutput`
```rust
pub struct OsMidiOutput(pub (crate) Arc<Mutex<Win32MidiAccess>>);
```

### `Win32MidiPort`
```rust
pub struct Win32MidiPort {
    handle: Win32MidiHandle,
    device_id: u32,
    desc: MidiPortDesc,
}
```

### `Win32MidiAccess`
```rust
pub struct Win32MidiAccess {
    input_senders: InputSenders,        // 输入事件发送者列表
    ports: Vec<Win32MidiPort>,
}
```

### 内部类型

**`InputSenders`**: `Arc<Mutex<Vec<mpsc::Sender<(MidiPortId, MidiData)>>>>` — 多路分发的输入通道集合

**`Win32MidiHandle`**:
```rust
enum Win32MidiHandle {
    Closed,
    OpenIn(HMIDIIN, *const InputProcContext),  // 输入设备句柄 + 回调上下文
    OpenOut(HMIDIOUT),                         // 输出设备句柄
}
```
实现 `Drop` 自动关闭 MIDI 句柄。

**`InputProcContext`**:
```rust
struct InputProcContext {
    senders: InputSenders,
    midi_port_id: MidiPortId,
}
```

## 关键函数/方法

### `Win32MidiAccess::new`
初始化 MIDI 系统，发送 `post_signal(Win32MidiInputsChanged)`。

### `Win32MidiAccess::update_port_list`
枚举所有 MIDI 输入和输出设备：
1. `midiInGetNumDevs` + `midiInGetDevCapsW` 枚举输入设备
2. `midiOutGetNumDevs` + `midiOutGetDevCapsW` 枚举输出设备
3. 使用 `get_unique_port_id` 生成稳定端口 ID（基于名称 + `wMid` + `wPid` + 序号）
4. 尝试复用旧端口句柄（通过端口 ID 匹配）

### `Win32MidiHandle::open_input`
1. 创建 `InputProcContext`（包含发送者通道）
2. 调用 `midiInOpen` 注册 `Win32MidiInputProc` 回调
3. 调用 `midiInStart` 启动输入

### `Win32MidiHandle::open_output`
调用 `midiOutOpen` 打开输出设备。

### `Win32MidiInputProc`（C 回调函数）
处理 MIDI 输入消息：
- `MM_MIM_DATA`: 提取 3 字节 MIDI 数据（`data0`、`data1`、`data2`），通过所有注册的 `InputSenders` 分发
- 其他消息：`MM_MIM_OPEN`、`MM_MIM_CLOSE`、`MM_MIM_LONGDATA`、`MM_MIM_ERROR`、`MM_MIM_LONGERROR`、`MM_MIM_MOREDATA` 为空操作

### `use_midi_inputs` / `use_midi_outputs`
启用/禁用指定端口。根据端口 ID 匹配打开或关闭对应句柄。

### `create_midi_input`
创建新的 `MidiInput` 实例（mpsc 接收器），注册发送者到 `input_senders`。

### `OsMidiOutput::send`
将 3 字节 MIDI 消息打包为 `u32` 短消息，调用 `midiOutShortMsg` 发送到所有匹配的输出端口。

## 平台集成

- 通过 `Cx::post_signal(live_id!(Win32MidiInputsChanged))` 通知主线程设备变更
- 在 `windows_media.rs` 中由 `CxMediaApi` 桥接：`midi_input`、`midi_output`、`use_midi_inputs`、`use_midi_outputs`
- 注意：当前代码中 `midi_reset` 存在 bug（`self.use_midi_inputs(&self, &[])` 应为 `self.use_midi_inputs(&[])`）
