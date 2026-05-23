# `checkbox.rs` — 复选框和开关

## 作用
实现 `CheckBox`（方块勾选框）和 `Toggle`（滑动开关）两个控件。

## 关键结构

### `CheckBox`
| 字段 | 类型 | 说明 |
|------|------|------|
| `checked` | `bool` | 是否选中 |
| `draw_bg` | `DrawQuad` | 背景方块 |
| `draw_check` | `DrawQuad` | 勾选符号 |
| `animator` | `Animator` | 动画状态机 |

### `Toggle`
| 字段 | 类型 | 说明 |
|------|------|------|
| `checked` | `bool` | 是否开启 |
| `draw_bg` | `DrawQuad` | 轨道背景 |
| `draw_handle` | `DrawQuad` | 滑动手柄 |
| `animator` | `Animator` | 动画状态机 |

## 方法详解

### `handle_event`（Widget）— CheckBox
- `FingerDown`：将 `pressed` 动画状态设为 `on`
- `FingerUp`：在本区域内释放时翻转 `checked` 值，触发 `CheckBoxAction::Changed` action

### `handle_event`（Widget）— Toggle
- 同 CheckBox，翻转 `checked` 值，触发 `ToggleAction::Changed`

### `draw_walk`（Widget）— CheckBox
- 根据 `checked` 状态绘制带颜色的背景方块和勾选标记

### `draw_walk`（Widget）— Toggle
- 根据 `checked` 值动画滑动 `handle`
- 手柄位置公式：`(self.width - self.handle_size) * self.checked_anim`

### `CheckBoxRef` 方法
- `get_checked` / `set_checked` / `set_text` / `changed`：状态访问
- `checked` 和 `unchecked` 事件检查

### `ToggleRef` 方法
- 同上，检查 `ToggleAction::Changed` 事件
