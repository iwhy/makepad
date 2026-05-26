# apple_game_input.rs — 游戏控制器输入

**文件路径:** `platform/src/os/apple/apple_game_input.rs`

**核心目的:** 使用 Apple GameController 框架处理游戏控制器输入。支持 Extended Gamepad、Micro Gamepad（Siri Remote）和 PlayStation/Xbox 无线控制器。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `GameControllerState` | 游戏控制器状态快照 |
| `ExtendedGamepadState` | 扩展手柄状态（摇杆、扳机、肩键、方向键） |
| `MicroGamepadState` | 微型手柄状态（Siri Remote） |
| `GameControllerHandler` | 控制器事件处理器，管理连接/断开回调和输入轮询 |

**`ExtendedGamepadState` 字段:**
- `left_thumbstick: Vec2` — 左摇杆 (x, y)
- `right_thumbstick: Vec2` — 右摇杆 (x, y)
- `left_trigger: f32` — 左扳机 (0.0–1.0)
- `right_trigger: f32` — 右扳机 (0.0–1.0)
- `button_a` / `button_b` / `button_x` / `button_y: bool` — 动作按钮
- `left_bumper` / `right_bumper: bool` — 肩键
- `dpad_up` / `dpad_down` / `dpad_left` / `dpad_right: bool` — 方向键
- `left_thumbstick_button` / `right_thumbstick_button: bool` — 摇杆按压

**`MicroGamepadState` 字段:**
- `dpad: Vec2` — 方向键/触摸表面 (x, y)
- `button_a: bool` — 确认按钮
- `button_x: bool` — 辅助按钮
- `pause: bool` — 暂停/菜单按钮

**关键方法:**
- `GameControllerHandler::new()` — 创建控制器处理器：
  - 注册 `GCControllerDidConnect` / `GCControllerDidDisconnect` 通知
  - 枚举已连接的控制器
- `GameControllerHandler::poll()` — 轮询所有控制器状态，返回 `Vec<GameControllerState>`
- `GameControllerHandler::start_polling(cb)` — 开始自动轮询并调用回调
- `GameControllerHandler::stop_polling()` — 停止轮询
- `GameControllerHandler::set_player_index(controller, index)` — 设置控制器玩家索引

**映射到 Makepad 虚拟键:**
- 控制器按钮映射到 Makepad `KeyCode` 枚举：
  - Button A → 按键 0, Button B → 按键 1, 依此类推
  - 摇杆和扳机映射为模拟轴事件
  - D-Pad 映射为方向键

**实现细节:**
- 使用 `GCController` 类的 `controllers` 类方法获取控制器列表
- 通过 `GCExtendedGamepad` / `GCMicroGamepad` 获取手柄状态
- 使用 `GCControllerDidConnectNotification` / `GCControllerDidDisconnectNotification` 监听连接变化
- 值归一化：摇杆和扳机值从 [-1,1] / [0,1] 归一化输出
- 死区处理：小值被归零以防止摇杆漂移
- Siri Remote 支持触摸表面和加速度计数据
- 振动反馈通过 `GCController` 的 `haptics` API

**平台集成:** macOS、iOS、tvOS，使用 GameController 框架
