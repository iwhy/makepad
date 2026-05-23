# keyboard_view.rs — 虚拟键盘适配视图

## 整体职能
`KeyboardView` widget 负责在虚拟键盘弹出/收起时自动调整内容区域的位置，确保输入框不被键盘遮挡。它监听系统键盘事件，计算键盘高度，并对内部的子 widget 应用垂直偏移动画。

## 主要数据结构
- **`KeyboardView`**：顶层 widget，包含 `content`（内容区域）、`draw_bg`（背景绘制）、`animator`（动画控制器）、`keyboard_rect`（键盘在屏幕空间中的矩形）、`content_offset`（当前内容偏移量）、`target_offset`（目标偏移量）。
- **`KeyboardViewAnim`**：`Animator` 派生结构，管理 `slide_up` 和 `slide_down` 动画状态。
- **`KeyboardViewWidgetRef`**：对 `KeyboardView` 的便捷访问封装，提供 `id!()` 宏支持。

## 方法与实现逻辑

### `fn script_component` — 脚本注册
注册 `KeyboardView` 和 `KeyboardViewAnim` 到脚本运行时。通过 `set_type_default` 定义默认样式，包括 `draw_bg` 的背景色和动画参数。

### `fn draw_walk` — 绘制与布局
1. 调用 `draw_bg.draw_abs(cx, rect)` 绘制背景。
2. 根据 `content_offset` 对内容区域应用垂直偏移变换：`cx.begin_rot_scale_translate(rect, 0.0, vec2(1.0, 1.0), vec2(0.0, content_offset))`。
3. 在偏移后的坐标空间中调用 `content.draw_walk(cx, scope, Walk::same())` 绘制子 widget。
4. 结束变换。

### `fn handle_event` — 事件处理
监听 `KeyboardEvent` 类型的系统事件：
- **`KeyboardEvent::Shown(height)`**：记录键盘高度到 `keyboard_rect`，计算 `target_offset` 为负值（上移），启动 `slide_up` 动画。
- **`KeyboardEvent::Hidden`**：将 `target_offset` 设回 0，启动 `slide_down` 动画。
同时处理 `Animator` 的动画帧事件，在每一帧中根据动画进度插值更新 `content_offset`。

### `fn handle_event_anim` — 动画帧处理
在 `KeyboardViewAnim` 的动画驱动下，每一帧计算 `content_offset` 的插值。使用 `AnimatorState` 定义的缓动曲线（默认 `Forward { duration: 0.3 }`），实现平滑的键盘跟随动画。当动画接近目标值时自动停止重绘请求。

### `fn keyboard_height` — 获取当前键盘高度
返回安全区域底部到键盘底部的距离，用于计算需要偏移的量。如果键盘未显示则返回 0。

### `fn content_offset_for_keyboard` — 计算内容偏移量
根据键盘高度和当前窗口大小，计算使输入框可见所需的垂直偏移量。考虑输入框在内容区域中的位置，确保偏移不会导致内容超出屏幕顶部。

### `fn reset` — 重置状态
清空 `keyboard_rect`、`content_offset` 和 `target_offset`，回到无键盘偏移的初始状态。
