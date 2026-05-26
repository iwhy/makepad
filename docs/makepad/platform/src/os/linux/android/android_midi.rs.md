# Android MIDI 实现

## 概述

`android_midi.rs` 实现了 Android 平台的 MIDI 输入/输出功能。使用 `amidi_sys` FFI 与 AMidi NDK API 交互。

**注意**：AMidi API 的输入/输出端口命名与直观理解相反（AMidi 的输出端口是接收数据的端口）。

## 核心类型

### `OsMidiOutput`
MIDI 输出接口，持有 `AndroidMidiAccess` 的 `Arc<Mutex<>>` 引用。

- `send(port_id, data)` — 发送 MIDI 数据到指定端口

### `OsMidiInput`
MIDI 输入接口，通过 `mpsc::Receiver` 接收消息。

- `receive()` — 从通道读取 MIDI 数据（会先调用 `read_inputs()` 轮询 AMidi）

### `AndroidMidiOutput` / `AndroidMidiInput`
底层 AMidi 端口包装。

- `AndroidMidiOutput` 包装 `AMidiInputPort`（发送端）
- `AndroidMidiInput` 包装 `AMidiOutputPort`（接收端）

两者操作互逆：
- 打开：`AMidiInputPort_open` / `AMidiOutputPort_open`
- 关闭：`AMidiInputPort_close` / `AMidiOutputPort_close`

### `AndroidMidiDevicePtr`
持有 AMidi 设备指针和端口描述列表。`release()` 调用 `AMidiDevice_release`。

### `AndroidMidiState`
状态机枚举：
- `OpenAllDevices` — 初始状态，请求枚举所有 MIDI 设备
- `OnErrorReload` — 出错需要重新加载
- `Ready` — 正常运行

### `AndroidMidiAccess`
MIDI 访问的核心管理结构。

## 关键方法

### `AndroidMidiAccess::new(change_signal)`
创建 `AndroidMidiAccess` 实例，初始状态为 `OpenAllDevices`。

### `read_inputs()`
轮询所有已打开的 AMidi 输出端口（接收端口），每次最多读取 384 字节（3 × 128），通过 `mpsc::Sender` 分发 MIDI 消息。

### `send_midi(port_id, data)`
发送 3 字节 MIDI 消息到指定输出端口。

### `use_midi_inputs(ports)` / `use_midi_outputs(ports)`
管理活动 MIDI 端口集合：
- 打开新出现的端口
- 关闭不再需要的端口
- 每个端口的 AMidi 端口只打开一次

### `get_updated_descs()`
根据状态返回端口描述：
- `OpenAllDevices`/`OnErrorReload`：返回 `None`（触发重新枚举）
- `Ready`：返回所有收集到的 `MidiPortDesc` 列表

### `midi_device_opened(name, java_device)`
从 Java 回调接收 MIDI 设备打开通知，创建 AMidi 设备并查询端口数量。

### `midi_disconnect()`
关闭所有端口、释放所有设备，清空状态。

## 实现说明

- 使用 `OnceLock` 缓存的 `ModuleLoader` 加载 `libamidi.so`（见 `amidi_sys.rs`）。
- MIDI 端口以延迟方式打开：只在 `use_midi_inputs`/`use_midi_outputs` 请求时才实际调 `AMidiXxxPort_open`。
- 错误处理：`AMidiOutputPort_receive` 返回负数时触发 `OnErrorReload` 状态重试。
- 头注释 "WARNING.. AMidi has inputs and outputs naming reversed" 是重要的使用提示。
