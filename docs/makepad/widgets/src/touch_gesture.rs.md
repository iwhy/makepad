# touch_gesture.rs — 触摸手势处理（滑动 & 弹跳动画）

## 整体职能
`TouchGesture` widget 为移动端触摸交互提供平滑的 **单指滑动（drag）** 和 **弹跳动画（fling/bounce）** 支持。它不直接渲染内容，而是作为手势状态机嵌入到需要触摸滚动的父 widget 中。

## 主要数据结构
- **`TouchGesture`**：持有触摸状态机，包括 `area`（触摸区域）、`current_state`（当前手势状态）、`velocity`（速度向量）、`drag_start_pos`（拖动起始位置）、`is_dragging` 标记等。
- **`TouchGestureState`**：枚举，取值 `Idle`（空闲）、`Dragging`（拖动中）、`Flinging`（惯性滑动中）。
- **`BounceAnim`**：`Animator` 派生的动画控制器，管理 `drag` 和 `release` 两个动画状态，控制弹簧效果的缓动曲线。
- **`Drag`**：记录单次触摸拖动的事件数据，包含 `start` / `current` / `delta` / `velocity` 信息。

## 方法与实现逻辑

### `fn script_component` — 脚本注册
注册 `TouchGesture` 和 `BounceAnim` 到脚本运行时。通过 `use mod.prelude.widgets_internal.*` 使用内部预导入。

### `fn handle_event` — 事件处理入口
接收 `Event`，区分 `TouchEvent`（触摸屏）和 `MouseEvent`（桌面鼠标模拟）。触摸事件通过 `hit_test` 判断是否命中手势区域，然后分别处理 `TouchDown` / `TouchMove` / `TouchUp` 三种触摸阶段。桌面鼠标事件类似，但使用鼠标按键代替触摸点。

### `fn handle_touch_down` — 触摸开始
记录触摸起始位置到 `drag_start_pos`。如果当前正在惯性滑动（Flinging），立即停止动画并捕捉当前速度作为初始速度。将状态切换为 `Dragging`。

### `fn handle_touch_move` — 触摸移动
计算当前触摸位置与上一帧位置的差值 `delta`，累计到 `total_drag`。更新 `velocity` 为指数移动平均（EMA）平滑后的瞬时速度。通过 `cx.request_animation_frame()` 触发持续重绘。冒泡 `TouchEvent::Dragging` 事件给父 widget。

### `fn handle_touch_up` — 触摸结束
根据最终速度 `velocity` 判断：如果速度超过阈值 `FLING_THRESHOLD`，进入 `Flinging` 状态并启动 `BounceAnim` 的 `release` 动画；否则直接回到 `Idle` 状态。冒泡 `TouchEvent::Released` 事件。

### `fn handle_event_anim` — 动画帧处理
每一帧根据 `BounceAnim` 的动画进度计算当前的偏移量。偏移量通过弹簧衰减函数计算：`offset = initial_velocity * decay_factor * sin(frequency * t) * e^(-damping * t)`。当动画接近静止（速度 < 0.1）时自动停止并回到 `Idle`。

### `fn draw_walk` — 绘制占位
不进行任何实际绘制，仅返回 `DrawStep::done()`。该 widget 只处理触摸手势逻辑，渲染由其他 widget 完成。

### `fn reset` — 重置手势状态
将 `current_state`、`velocity`、`drag_start_pos`、`is_dragging` 等字段恢复为默认值。

### `fn hit_test` — 命中检测
判断给定的屏幕坐标是否位于 `self.area` 所定义的矩形区域内。如果未初始化则返回 `false`。
