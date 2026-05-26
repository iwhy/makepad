# `tooltip.rs` — 工具提示

## 作用
实现一个跟随鼠标位置的悬浮提示框，可自定义文本和位置。用户交互（点击、滚动、触摸）时自动隐藏。

## 关键结构

### `Tooltip`
| 字段 | 类型 | 说明 |
|------|------|------|
| `view` | `View`（`#[deref]`） | 内部 View |
| `draw_list` | `Option<DrawList2d>` | 叠加层绘制列表 |
| `draw_bg` | `DrawQuad` | 背景绘制 |
| `opened` | `bool` | 是否显示 |
| `tooltip_pos` | `Vec2d` | 工具提示在屏幕上的位置 |

## 方法详解

### `handle_event`（Widget）
- 仅在 `opened == true` 时处理事件。先向内部 content 转发事件
- 捕获以下事件类型并自动隐藏：`BackPressed`、`MouseDown`、`MouseUp`、`Scroll`，以及 Touch 开始事件
- 不关心命中细节，只关心"发生了交互"这一事实

### `draw_walk`（Widget）
- 使用 `draw_list.begin_overlay_reuse` 叠加绘制
- 获取当前 pass 尺寸，启动根级海龟布局，先绘制 `draw_bg`
- 仅在 `opened` 时绘制内容，使用 `self.tooltip_pos` 作为绝对位置偏移

### `set_text`（Widget）
- 通过 `ids!(content.tooltip_label)` 路径找到内部 Label 控件，设置提示文本

### `set_pos`
- 设置工具提示的屏幕坐标位置

### `show`
- 设置 `opened = true`。重绘 `draw_list` 确保首次调用时可见（因为 View 的 area/draw_list 在首次绘制前尚未初始化）
- 同时调用 `self.redraw(cx)`

### `show_with_options`
- 一键设置位置、文本并显示

### `hide`
- 设置 `opened = false`，重绘 `draw_list` 和自身

### `TooltipRef` 方法
- `set_text` / `set_pos` / `show` / `show_with_options` / `hide`：通过 `borrow_mut` 安全操作内部状态
