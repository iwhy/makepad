# slides_view.rs — 幻灯片视图

## 整体职能
`SlidesView` widget 实现一个 **全屏幻灯片/轮播** 视图，支持在多个页面之间水平滑动切换。常用于引导页、图片轮播、或分步向导。

## 主要数据结构
- **`SlidesView`**：顶层 widget，包含 `pages`（页面列表）、`current_index`（当前页索引）、`draw_dots`（页码指示器绘制）、`animator`（翻页动画控制器）、`touch_gesture`（触摸手势处理器）等字段。
- **`SlidePage`**：表示一页幻灯片，包含 `content`（widget 内容）、`background`（可选背景色）、`transition`（翻页过渡效果枚举）。
- **`SlideTransition`**：枚举，取值 `Slide`（水平滑动）、`Fade`（淡入淡出）、`Zoom`（缩放）。
- **`SlidesViewAction`**：枚举，包括 `PageChanged(from, to)`、`PageSwipeComplete(index)`。

## 方法与实现逻辑

### `fn script_component` — 脚本注册
注册 `SlidesView`、`SlideTransition` 等类型到脚本运行时。定义默认指示器样式（小圆点，活跃态 8px，非活跃态 6px，间距 12px）。

### `fn draw_walk` — 幻灯片绘制
1. 根据 `current_index` 和 `anim_offset`（动画插值偏移）计算当前页和目标页的屏幕位置。
2. 根据 `transition` 类型执行对应的过渡效果：
   - **Slide**：两页水平排列，根据 `anim_offset` 在 X 轴上平移。
   - **Fade**：当前页逐渐透明，目标页逐渐显现。
   - **Zoom**：当前页缩小的同时目标页放大。
3. 调用 `draw_dots.draw_abs(cx, dots_rect)` 在底部绘制页码指示点。
4. 支持可选的自动播放模式（`auto_play` + `auto_play_interval`）。

### `fn handle_event` — 事件处理
- **触摸滑动**：通过 `TouchGesture` 检测水平方向的快速滑动或拖拽。滑动超过阈值（宽度 30%）或速度超过阈值时翻页。
- **键盘事件**：左右方向键翻页。
- **定时器**：`auto_play` 模式下，定时触发 `next_page()`。

### `fn go_to_page` — 跳转到指定页
更新 `current_index` 到目标页（范围检查 0..pages.len()）。启动翻页动画。触发 `SlidesViewAction::PageChanged`。

### `fn next_page` / `prev_page` — 翻页
`next_page`：索引加一，如果已在最后一页则根据 `loop_enabled` 决定回到第一页或停止。
`prev_page`：索引减一，第一页时回到最后一页（循环模式）。

### `fn add_page` — 添加新页面
向 `pages` 列表末尾添加一个 `SlidePage`。支持通过脚本表达式动态构建页面内容。

### `fn remove_page` — 移除页面
从指定索引移除页面。如果移除的是当前页或之前的页面，适当调整 `current_index`。

### `fn set_auto_play` — 设置自动播放
开启或关闭自动轮播。开启时启动一个定时器，每隔 `auto_play_interval` 毫秒自动调用 `next_page()`。当用户手动交互时自动暂停自动播放。

### `fn update_indicator_dots` — 更新指示器
根据 `pages.len()` 和 `current_index` 计算每个指示点的大小和活跃状态。活跃点放大且使用强调色，非活跃点使用半透明色。
