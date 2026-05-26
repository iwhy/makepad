# Linux 原始输入设备

## 概述

`raw_input.rs` 实现了 Linux 原始输入设备（`/dev/input/event*`）的轮询和事件解析。支持键盘、鼠标、触摸屏等多种输入设备，将其转换为 Makepad 平台事件。

## 枚举定义

### 输入事件码枚举

| 枚举 | 说明 | 覆盖范围 |
|------|------|----------|
| `InputEventType` | 输入事件类型：`EV_SYN`、`EV_KEY`、`EV_REL`、`EV_ABS` 等 |
| `EvSynCodes` | 同步码：`SYN_REPORT`、`SYN_DROPPED` |
| `EvKeyCodes` | Linux 键码（0-248） + BTN 码（0x100-0x1BF） |
| `EvRelCodes` | 相对轴：`REL_X`、`REL_Y`、`REL_WHEEL` 等 |
| `EvAbsCodes` | 绝对轴：`ABS_X`、`ABS_Y`、`ABS_MT_*` 多点触摸 |
| `EvMscCodes` | 杂项码 |
| `EvSwCodes` | 开关码 |
| `EvLedCodes` | LED 码 |
| `EvSndCodes` | 声音码 |
| `EvRepCodes` | 重复码 |
| `KeyAction` | 按键动作：`KeyUp = 0`、`KeyDown = 1`、`KeyRepeat = 2` |

### `InputEvent`
原始输入事件结构体（匹配 `struct input_event`）：
- `time` — `timeval` 时间戳
- `ty` — 事件类型（`InputEventType`）
- `code` — 事件码
- `value` — 事件值

## 核心类型

### `RawInput`
原始输入管理器：
- `modifiers` — 当前修饰键状态
- `receiver` — 输入事件通道接收端
- `width` / `height` — 屏幕尺寸（用于绝对坐标归一化）
- `dpi_factor` — DPI 缩放因子
- `abs` — 当前鼠标/触摸绝对位置

## 关键方法

### `new(width, height, dpi_factor)`
初始化输入系统：
- 尝试打开 `/dev/input/event0` 到 `/dev/input/event11`
- 每个设备启动一个独立线程轮询原始 `input_event` 结构（16 字节）
- 事件通过 `mpsc::Sender` 发送到主通道

### `poll_raw_input(time, window_id) -> Vec<DirectEvent>`
轮询所有输入事件：
1. 从通道读取事件，直到为空
2. 累积 `EV_KEY`、`EV_REL`、`EV_ABS` 事件
3. 在 `SYN_REPORT` 时批量处理
4. `SYN_DROPPED` 处理：清空缓冲区并跳过损坏数据

### `process_rel_event` / `process_abs_event`
处理鼠标/触摸位置更新，生成 `DirectEvent::MouseMove`。

### `process_key_event`
核心按键处理（约 200+ 键码映射）：
- **按键按下**：生成 `KeyDown`（文本输入在无修饰键时额外生成 `TextInput`），更新修饰键状态，鼠标/触摸按钮生成 `MouseDown`
- **按键抬起**：生成 `KeyUp`，更新修饰键状态，鼠标/触摸按钮生成 `MouseUp`
- **按键重复**：生成 `KeyDown(is_repeat: true)` 和 `TextInput`

## 键码映射

`EvKeyCodes` 到 `KeyCode` 的转换覆盖了标准键盘的所有键：
- 字母 A-Z、数字 0-9
- 功能键 F1-F24
- 方向键、Home/End/PgUp/PgDn
- 小键盘（0-9 及运算符）
- 修饰键（Ctrl/Shift/Alt/Logo）
- 鼠标按钮（BTN_LEFT/RIGHT/MIDDLE/SIDE/EXTRA）
- 多媒体键（音量、播放、暂停等）
- 未识别键码映射到 `KeyCode::Unknown`

## 实现说明

- 使用 `mpsc::channel` 实现跨线程事件传递。
- 绝对坐标（`ABS_MT`）通过 `dpi_factor` 缩放，相对坐标（`REL`）累加后裁剪到屏幕范围。
- 触摸屏的 `BTN_TOUCH` 事件映射为鼠标主按钮。
- 文本输入通过 `key_code.to_char(shift)` 函数生成对应字符。
- `SYN_DROPPED` 处理确保在输入缓冲区溢出时数据一致性。
- 每个设备独立线程的生命周期由 `mpsc::Sender` 引用控制（发送端释放后线程 `recv()` 会返回错误，线程退出）。
