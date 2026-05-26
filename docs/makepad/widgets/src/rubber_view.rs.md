# rubber_view.rs — RubberView 弹性动画视图组件

## 文件概述

`RubberView` 是一种具有弹性动画效果的 View 组件，支持通过拖拽触发关闭操作（如移动端的下拉刷新、消息面板滑动关闭）。核心机制是 `ElasticAnimation` — 一个带弹性回弹效果的动画值系统。

---

## `RubberView` 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct RubberView {
    #[source] source: ScriptObjectRef,
    #[deref] view: View,                     // 继承 View 的布局和绘制能力
    #[live] animator: Animator,
    #[live] animate: ElasticAnimation,       // 弹性动画状态
    #[rust] last_mouse_pos: DVec2,           // 上次鼠标位置（用于拖拽增量）
    #[live] max_velocity: f64,               // 最大拖拽速度
    #[live] elastic_border_color: Vec4f,     // 弹性变形时的边界颜色
}
```

### ElasticAnimation

```rust
pub struct ElasticAnimation {
    value: f64,             // 当前动画值（如弹性偏移量）
    velocity: f64,          // 当前速度
    damping: f64,           // 阻尼系数（控制回弹速度）
    stiffness: f64,         // 弹性系数（控制回弹力度）
    threshold: f64,         // 触发阈值（超过此值时触发关闭动作）
    is_dragging: bool,
    is_active: bool,
}
```

弹性动画数学实现（类似弹簧-质点系统）：

```rust
fn update(&mut self, dt: f64) {
    // 1. 计算弹性力：F = -k * x（胡克定律）
    let spring_force = -self.stiffness * self.value;
    
    // 2. 计算阻尼力：F_d = -c * v
    let damping_force = -self.damping * self.velocity;
    
    // 3. 更新加速度、速度、位置
    let acceleration = spring_force + damping_force;
    self.velocity += acceleration * dt;
    self.value += self.velocity * dt;
    
    // 4. 如果值和速度都接近零，停止动画
    if self.value.abs() < EPSILON && self.velocity.abs() < EPSILON {
        self.is_active = false;
    }
}
```

---

## 拖拽与关闭流程

```
MouseDown → 记录初始位置
    ↓
MouseMove → 计算增量 → 更新 ElasticAnimation.value
    ↓
MouseUp → 检查 value 是否超过 threshold
    ├─ 否 → 弹性回弹（animation 自然衰减到 0）
    └─ 是 → 触发关闭动作（发出 Action）
```

### 鼠标交互

1. `handle_event` 在 MouseDown 时记录 `last_mouse_pos` 并设置 `is_dragging = true`。
2. MouseMove 时计算 `dx` / `dy`（根据配置的方向），更新 `animate.value += dx`。
3. 同时更新速度 `animate.velocity = dx / dt`，用于惯性效果。
4. MouseUp 时检查 `threshold`：
   - 未超过：释放弹性动画，`animate` 自然回弹到 0。
   - 超过：触发关闭事件，发出 `WidgetAction`。

### 绘制

draw_walk 时根据 `animate.value` 调整内容的偏移量：

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    // 应用弹性偏移到子组件绘制
    cx.turtle().set_offset(DVec2::new(0.0, self.animate.value));
    // 如果处于弹性变形状态，改变 draw_bg 颜色
    // 绘制完成后还原偏移
}
```

---

## 使用场景

- **滑动关闭面板**：向右滑动关闭通知/消息面板。
- **下拉刷新**：下拉超过阈值触发刷新动作。
- **弹性边界**：滚动到边缘时产生弹性拉伸效果。
- **卡片滑动**：Tinder 风格左滑/右滑操作。

---

## 参数调节

| 参数 | 说明 | 典型值 |
|------|------|--------|
| `damping` | 阻尼系数，越大回弹越快停止 | 8.0–12.0 |
| `stiffness` | 弹性系数，越大回弹越硬 | 100.0–300.0 |
| `threshold` | 触发阈值（像素） | 100.0–200.0 |
| `max_velocity` | 最大速度，限制拖拽过快 | 2000.0 |
