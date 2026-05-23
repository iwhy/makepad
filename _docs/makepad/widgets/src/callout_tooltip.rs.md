# callout_tooltip.rs — 带三角箭头的气泡提示

## 整体职能
`CalloutTooltip` widget 实现一个带 **指向箭头（callout arrow）** 的浮动提示气泡，常用于表单字段的验证错误提示、上下文帮助信息或功能引导。箭头自动指向触发元素，支持四种方位（上/下/左/右）。

## 主要数据结构
- **`CalloutTooltip`**：顶层 widget，包含 `draw_bg`（气泡背景）、`draw_arrow`（箭头绘制）、`draw_text`（文本绘制）和布局/定位属性。
- **`DrawCalloutArrow`**：自定义 Draw shader，`#[repr(C)]` 布局，包含 `arrow_size`（箭头大小）、`arrow_offset`（箭头偏移量）、`arrow_angle`（箭头旋转角度）等实例属性。
- **`CalloutPosition`**：枚举，定义箭头指向方位：`Top`（箭头朝上，气泡在目标下方）、`Bottom`（箭头朝下）、`Left`（箭头朝左）、`Right`（箭头朝右）。含 `Auto` 模式根据可用空间自动选择最佳方位。
- **`CalloutTheme`**：`#[live]` 属性集合，包括 `color_bg`（背景色）、`color_border`（边框色）、`color_text`（文本色）、`border_radius`（圆角）、`padding`（内边距）。

## 方法与实现逻辑

### `fn script_component` — 脚本注册
注册 `CalloutTooltip`、`DrawCalloutArrow` 以及 `CalloutPosition` 枚举到脚本运行时。定义默认主题：蓝灰色背景、中灰文本、6px 圆角。

### `fn show` — 显示提示
接收目标矩形 `target_rect` 和可选的显示文本。计算最佳位置：
1. 检查目标四周的可用空间。
2. 选择空间最大的方位（`Auto` 模式），或使用指定的方位。
3. 根据方位设置 `arrow_offset`（箭头在气泡边缘的位置）和 `arrow_angle`（箭头旋转角度）。
4. 将气泡定位在目标附近，应用 `gap`（箭头与目标之间的距离）。
5. 通过 `cx.request_redraw()` 触发显示。

### `fn hide` — 隐藏提示
将可见性标记设为 `false`。支持可选的淡出动画（通过 `Animator`）。

### `fn draw_walk` — 气泡绘制
1. 如果不可见，返回 `DrawStep::done()`。
2. 根据 `CalloutPosition` 计算气泡矩形的位置，使其紧贴目标元素并避开屏幕边缘。
3. 调用 `draw_bg.draw_abs(cx, bubble_rect)` 绘制圆角背景矩形（气泡主体）。
4. 调用 `draw_arrow.draw_abs(cx, arrow_rect)` 在指定位置绘制指向箭头三角形。
5. 调用 `draw_text.draw_walk(cx, scope, text_walk)` 绘制文本内容。

### `fn handle_event` — 事件处理
监听全局鼠标事件。当用户点击气泡外部区域时自动调用 `hide()`。支持通过 `dismiss_delay`（自动消失延迟）实现定时关闭。

### `fn set_text` — 设置提示文字
更新要显示的文本内容。如果气泡当前可见，触发重绘以反映文本变化。

### `fn reposition` — 重新定位
当触发元素位置变化时（例如滚动或窗口大小改变），重新计算气泡的位置。通常在父容器的滚动事件或窗口 resize 事件中被调用。

### `fn calculate_callout_rect` — 计算气泡与箭头矩形
根据选定的方位和箭头大小，返回气泡主体矩形和箭头矩形的具体坐标。箭头绘制为一个等腰三角形，其底边与气泡边缘对齐。

### `fn choose_best_position` — 自动选择最佳方位
遍历 `Top` / `Bottom` / `Left` / `Right` 四个方位，计算每个方位下气泡可用的像素空间，返回空间最大的方位。考虑屏幕边缘的安全边距。
