# `game_input.rs` — 游戏输入事件系统

## 概述

该文件定义了 Makepad 的游戏手柄（Gamepad）和方向盘（Wheel）输入事件类型及相关数据结构。支持设备的连接/断开通知，以及完整的游戏输入状态查询。

---

## 设备连接事件

### `GameInputConnectedEvent` 枚举
- `Connected(GameInputInfo)` — 设备已连接
- `Disconnected(GameInputInfo)` — 设备已断开

### `GameInputInfo`
- `id: LiveId` — 设备唯一标识
- `name: String` — 设备名称

---

## 输入状态枚举

### `GameInputState`
- `Gamepad(GamepadState)` — 游戏手柄状态
- `Wheel(WheelState)` — 方向盘状态

---

## `GamepadState`

标准游戏手柄的完整状态：

| 字段 | 类型 | 描述 |
|------|------|------|
| `a` / `b` / `x` / `y` | `f32` | 面部按钮（ABXY），0.0~1.0 |
| `left_shoulder` / `right_shoulder` | `f32` | 肩部按钮（LB/RB） |
| `left_trigger` / `right_trigger` | `f32` | 模拟扳机（LT/RT） |
| `select` / `start` / `home` | `f32` | 系统按键 |
| `left_thumb` / `right_thumb` | `f32` | 摇杆按压 |
| `dpad_up` / `down` / `left` / `right` | `f32` | 方向键 |
| `left_stick` / `right_stick` | `Vec2` | 模拟摇杆轴 |

所有按钮值使用 `f32`（0.0 = 未按下，1.0 = 完全按下），支持压感检测。

---

## `WheelState`

赛车方向盘状态：
- `steering: f32` — 转向角度
- `throttle: f32` — 油门
- `brake: f32` — 刹车
- `clutch: f32` — 离合器
- `steer_force: f32` — 力反馈力度

---

## `GameInputEventChannel`

使用 `std::sync::mpsc::channel` 实现的线程安全事件通道：
- `sender: Sender<GameInputConnectedEvent>` — 发送端
- `receiver: Receiver<GameInputConnectedEvent>` — 接收端
- `Default` 实现自动创建新的通道对

设计意图：游戏输入设备的连接/断开是异步事件，使用 MPSC 通道可以在后台线程中提交事件，主线程在事件循环中消费。
