# `radio.rs` — 单选按钮和组合

## 作用
实现 `RadioButton`（单个单选按钮）和 `RadioGroup`（单选按钮组），支持自动互斥选择。

## 关键结构

### `RadioButton`
| 字段 | 类型 | 说明 |
|------|------|------|
| `checked` | `bool` | 是否选中 |
| `value` | `u64` | 关联值 |
| `unique_id` | `LiveId` | 控件 ID |
| `draw_bg` | `DrawQuad` | 外部圆圈 |
| `draw_dot` | `DrawQuad` | 内部选中圆点 |
| `animator` | `Animator` | 动画状态机 |

### `RadioAction`
| 变体 | 说明 |
|------|------|
| `Pressed { value: u64, clicked: bool }` | 选中事件 |
| `None` | 无操作 |

### `RadioGroup`
- 外壳 `View`，内部包含多个 `RadioButton`

## 方法详解

### `handle_event`（Widget）— RadioButton
- `FingerUp`：翻转 `checked`，触发 `RadioAction::Pressed` action，包含按钮的 `value` 和 `clicked` 标志

### `draw_walk`（Widget）
- 绘制半透明背景 → 内容（空白或 items）→ 选中时绘制 highlight

### `RadioGroup` 的互斥逻辑
- `RadioGroup` 在 `handle_event` 中监听 `RadioAction::Pressed` 事件
- 当任一 RadioButton 被按下时，遍历组内所有按钮，只将点击的那个设为 checked，其余清空 checked
- 生成 `RadioAction::Pressed` action（`clicked = false`）透传出去

### `RadioButtonRef` 方法
- `pressed`：检查 actions 中是否有该按钮被按下
- `set_value`：设置关联值
- `set_checked` / `get_checked` / `set_text`：状态访问

### `RadioGroupRef` 方法
- `pressed_value`：检查 actions 中是否有组内按钮被按下并返回其值
- `set_selected`：按值设置选中项（未匹配则清除所有选中）
- `get_selected_value`：获取当前选中值
