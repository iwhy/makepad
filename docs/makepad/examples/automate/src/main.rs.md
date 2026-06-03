# automate/src/main.rs

演示 Makepad 的硬件控制应用 — Art-Net DMX512 灯光控制器，支持 MIDI 输入、场景预设和实时 UI 反馈。

## 整体结构

### UI 组件模板（第33-181行）

- `DeckFrame` / `SectionFrame`：深色主题的圆角容器
- `ScenePad`：自定义外观的场景按钮
- `TransportToggle`：自定义外观的开关按钮
- `EncoderKnob`：旋钮控件（`Rotary`)
- `APCFader`：完全自定义 Shader 的推子（带轨道、填充条、滑块帽的 3D 效果垂直滑块）

### UI 布局（第182-388行）

控制台布局包含三个区域：
1. **ENCODERS**（第204-221行）：8 个旋钮
2. **TRANSPORT**（第223-258行）：3 行运输控制按钮
3. **SCENES**（第262-290行）：13 个场景按钮
4. **FADERS**（第292-383行）：8 个推子 + 主推子

### DMX 引擎（第391-661行）

- `ControllerState` / `ControllerButtons`：控制器状态
- `MidiMirrorState`：MIDI 镜像状态（CC 值和音符值）
- `UiControlMessage`：UI → 引擎线程的消息枚举
- `apply_dmx_mapping`：将控制器状态映射到 DMX 512 协议输出
- `preset_file` / `load_state_file` / `save_state_file`：场景预设文件的读写（RON 格式）
- `dmx_u8` / `dmx_f32` / `map_wargb`：DMX 协议辅助函数

### 后台线程（第755-1060行）

`start_dmx_bridge` 启动两个线程：
1. **MIDI/UI 主循环**（第762-1058行）：
   - 接收 UI 控制消息（旋钮/推子/场景/运输）
   - 接收 MIDI 输入事件
   - 每秒 44 帧生成 DMX 数据包并通过 UDP 广播
   - 每秒保存一次当前状态到文件
2. Audio callback 配置（auto channel）

### Rust 侧事件处理（第1063-1159行）

- `handle_startup`：启动 DMX 桥接线程并刷新 UI
- `handle_midi_ports`：扫描和显示 MIDI 设备
- `handle_signal`：从后台线程接收状态快照更新 UI
- `handle_actions`：将 UI 控件事件转发到后台线程

## 关键 API

- `Rotary` widget — 旋转编码器 UI
- `Slider` + 自定义 `pixel` 着色器 — 完全自定义外观的垂直推子
- `UdpSocket` — UDP 广播发送 Art-Net 协议
- `MidiInput` / `MidiEvent` — MIDI 输入处理
- `FromUISender` / `ToUIReceiver` — UI ↔ 后台线程跨线程通信
- `SerRon` / `DeRon` — RON 序列化用于预设文件
