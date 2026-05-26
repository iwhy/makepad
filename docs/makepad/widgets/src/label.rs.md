# `label.rs` — 文本标签系列

## 作用
实现 `Label`（普通标签）、`H1`/`H2`/`H3`（标题）、`LinkLabel`（超链接标签）等文本显示控件。

## 脚本 DSL

```
Label     { text: "Body text",     font_size: theme.font_size_p }
H1        { text: "Heading 1",     font_size: theme.font_size_h1 }
H2        { text: "Heading 2",     font_size: theme.font_size_h2 }
H3        { text: "Heading 3",     font_size: theme.font_size_h3 }
LinkLabel { text: "Clickable link" }
```

## 关键结构

### `Label`
| 字段 | 类型 | 说明 |
|------|------|------|
| `draw_bg` | `DrawQuad` | 可选背景 |
| `draw_text` | `DrawText` | 文本绘制 |
| `text_walk` | `Walk` | 覆盖 walking walk |

### `LinkLabel`
| 字段 | 继承自 Label + | 说明 |
|------|---------------|------|
| `draw_underline` | `DrawText` | 下划线绘制 |

## 方法详解

### `handle_event`（Widget）— Label
- 空实现，标签不交互

### `handle_event`（Widget）— LinkLabel
- `FingerDown`：在区域内按下时设置 `pressed = true`，播放按下动画
- `FingerUp`：在区域内释放时触发 `LinkLabelAction::Pressed`
- `FingerHoverIn/FingerHoverOut`：控制悬浮动画（下划线展开/收起）

### `draw_walk`（Widget）— Label
- 使用 `text_walk` 覆盖默认 walk。背景优先，文本在内容区域居中绘制

### `draw_walk`（Widget）— LinkLabel
- 与 Label 类似，额外在底部根据 `animator` 动画绘制下划线（从中间向两侧展开）

### `LinkLabelRef` 方法
- `pressed`：检查 actions 中是否有链接被点击
- `set_text`：设置文本并重绘

### `LinkLabel` 动画
- `active_cursor`：悬浮时 `hover` 动画触发下划线 `draw_underline` 的 `w` 参数从 0 到 1 展开
- `link_text.transition`：文本颜色过渡动画
