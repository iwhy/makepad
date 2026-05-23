# `button.rs` — 按钮系列控件

## 作用
实现 Makepad 的三个按钮变体：

| 控件 | 脚本定义位置 | 视觉风格 |
|------|-------------|---------|
| `Button` | `mod.widgets` | 标准按钮（有背景、边框） |
| `ButtonFlat` | `mod.widgets` | 扁平按钮（只有文本） |
| `ButtonFlatter` | `mod.widgets` | 超扁平按钮（更简洁） |

## 方法详解

### `handle_event`（Widget）
- `FingerDown`：设置 `pressed = true`，应用按下视觉效果（通过 `Animator` 的 `pressed` 状态）
- `FingerUp`：根据是否在按钮区域内决定触发 `ButtonAction::Pressed`
- `FingerHoverIn/FingerHoverOut`：控制 `hover` 动画状态
- 如果 `opacity < 0.01`，按钮不可交互（`hit_test = false`）
- 支持通过 `Animator` 实现按下/悬浮/停用状态的平滑过渡

### `draw_walk`（Widget）
- 若 `pressed`，按下时在文本位置添加像素级偏移（`text_shift`）
- 依次绘制背景四边形和文本
- 可选绘制边框

### `ButtonRef` 方法
- `pressed(actions)`：检查指定按钮是否在 actions 中被按下
- `is_hovered`：检查悬浮状态
- `set_text`：动态设置按钮文本

### `ButtonSet` 方法
- `pressed(actions)`：批量检查多个按钮的按下事件，返回 `((id1, btn1), (id2, btn2))` 元组

### 脚本 DSL
```
Button { text: "Click" }     // 标准圆角背景按钮
ButtonFlat { text: "Click" } // 仅文本按钮
ButtonFlatter { text: "Click" } // 更精简的文本按钮
```
