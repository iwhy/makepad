# windows_game_input.rs — 游戏输入接口（XInput + DirectInput）

**文件路径**: platform/src/os/windows/windows_game_input.rs (598行)
**核心用途**: 通过 XInput 轮询 Xbox 手柄状态，通过 DirectInput 8 轮询赛车方向盘等游戏控制器，支持力反馈。

## 常量

- `GUID_ConstantForce: GUID = 0x13541C20_8E33_11D0_9AD0_00A0C9A06E35` — 力反馈常量力 GUID
- `DIDFT_OPTIONAL: u32 = 0x80000000` — 可选轴标记
- 全局 `DF_JOYSTICK2_FORMAT: Option<(DIDATAFORMAT, Vec<DIOBJECTDATAFORMAT>)>` — DIJOYSTATE2 数据格式缓存

## 结构体

### `WindowsGameInput`
```rust
pub struct WindowsGameInput {
    pub gamepads: Vec<GameInputInfo>,          // 设备信息列表
    pub states: Vec<GameInputState>,           // 对应设备状态
    pub direct_input: Option<IDirectInput8W>,  // DirectInput 接口
    pub di_devices: Vec<(LiveId, IDirectInputDevice8W, GUID, Option<IDirectInputEffect>)>, // DI 设备
    pub next_wheel_id: u64,                   // 方向盘 ID 计数器
    pub enum_timer: u64,                       // 枚举计时器（每 200 帧枚举一次）
}
```

## 关键函数

### `WindowsGameInput::poll`

两级轮询架构：

#### 1. XInput 轮询（Xbox 手柄, 端口 0-3）
- 调用 `XInputGetState(i, &state)` 检查控制器连接状态
- 解析按钮：`dpad_up/down/left/right`、`start/select`、`left/right_thumb`、`left/right_shoulder`、`a/b/x/y`
- 解析扳机：`left_trigger` / `right_trigger` (0-255 → 0.0-1.0)
- 解析摇杆：`left_stick` / `right_stick` (-32768-32767 → -1.0-1.0)
- 设备 ID: `LiveId(i as u64)`（XInput 设备 < 128）
- 状态类型: `GameInputState::Gamepad`

#### 2. DirectInput 轮询（赛车方向盘等）
- 每 200 帧通过 `EnumDevices(DI8DEVCLASS_GAMECTRL, DIEDFL_ATTACHEDONLY)` 枚举设备
- 过滤 `DI8DEVTYPE_DRIVING` 类型设备
- 对新设备：
  1. `CreateDevice` → `SetDataFormat` (DIJOYSTATE2)
  2. `SetCooperativeLevel(DISCL_BACKGROUND | DISCL_EXCLUSIVE)`（力反馈需要独占模式）
  3. 创建常量力反馈效果 `IDirectInputEffect`
  4. 设备 ID: `LiveId(next_wheel_id++)`（≥ 128）
  5. 状态类型: `GameInputState::Wheel`
- 轮询已连接的 DI 设备：
  - `Poll()` → `GetDeviceState` 获取 `DIJOYSTATE2`
  - 映射轴：`lX` → 转向、`lY` → 油门（取反）、`lRz` → 刹车、`rglSlider[0]` → 离合
  - 力反馈更新：根据 `steer_force` 调用 `SetParameters(DIEP_TYPESPECIFICPARAMS | DIEP_START)` 更新常量力大小

### `ensure_data_format_initialized`

单次初始化 `DIDATAFORMAT` 用于 `DIJOYSTATE2`：
- 8 个模拟轴（X、Y、Z、Rx、Ry、Rz、Slider×2）
- 4 个 POV（方向键）
- 128 个按钮
- 所有轴标记为 `DIDFT_OPTIONAL`（允许设备缺少某些轴）

### 辅助函数
- `normalize_axis(i16) -> f32`: 将轴值 (-32768..32767) 映射到 (-1.0..1.0)
- `norm_axis(i32) -> f32`: 将 32 位轴值映射到 (-1.0..1.0)
- `norm_trig(i32) -> f32`: 将踏板值 (0..65535) 映射到 (0.0..1.0)

## 平台集成

- 在 `windows.rs` 的 `handle_game_input_events` 中每帧调用 `poll()`
- 设备连接/断开通过 `GameInputConnectedEvent` 事件通道传递
- 通过 `Cx::handle_repaint` 中的 `Signal` 事件触发状态更新
- `CxGameInputApi` trait 提供 `game_input_state`、`game_input_states`、`game_input_state_mut`、`game_input_states_mut` 方法
