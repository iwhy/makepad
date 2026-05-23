# scroll_bar.rs — ScrollBar 滚动条组件

## 文件概述

`ScrollBar` 是 Makepad 的滚动条控件，支持横向（X）和纵向（Y）滚动。它处理鼠标滚轮事件、拖拽滚动、惯性滑动和动画滚动。

---

## `ScrollBar` 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct ScrollBar {
    #[source] source: ScriptObjectRef,
    #[walk] walk: Walk,
    #[layout] layout: Layout,
    #[redraw] #[live] draw_bg: DrawQuad,        // 滚动条轨道背景
    #[redraw] #[live] draw_thumb: DrawQuad,      // 滑块（thumb）
    #[live] animator: Animator,                  // 动画引擎
    #[live] orientation: Axis,                   // 方向：X 或 Y
    #[live] always_show_thumb: bool,             // 始终显示滑块
    #[live] scroll_to_key: Option<LiveId>,       // 滚动到指定 Widget（键盘导航）
    #[rust] scroll_pos: f64,                     // 当前滚动位置
    #[rust] content_size: f64,                   // 内容总尺寸
    #[rust] view_size: f64,                      // 可视区域尺寸
    #[rust] max_scroll_pos: f64,                 // 最大可滚动位置
    #[rust] scroll_velocity: f64,                // 滚动惯性速度
    #[rust] is_dragging: bool,
    #[rust] thumb_start_pos: f64,                // 拖拽开始时滑块位置
}
```

---

## 核心逻辑

### 滚动位置计算

```rust
self.max_scroll_pos = (self.content_size - self.view_size).max(0.0);
```

- 当 `content_size <= view_size` 时，`max_scroll_pos = 0.0`，无需滚动。
- `scroll_pos` 被约束在 `[0.0, max_scroll_pos]` 范围内。

### 滑块尺寸和位置

```rust
let thumb_size = (view_size / content_size) * view_size;
let thumb_pos = (scroll_pos / max_scroll_pos) * (view_size - thumb_size);
```

- 滑块大小反映可见内容占全部内容的比例。内容越多，滑块越小。
- 滑块位置反映当前滚动进度。

### 鼠标滚轮

```rust
fn handle_scroll(&mut self, cx, event) {
    let delta = event.scroll_delta(self.orientation);  // Y 方向或 X 方向
    self.scroll_pos = (self.scroll_pos - delta * scroll_speed)
        .clamp(0.0, self.max_scroll_pos);
    self.scroll_velocity = -delta * scroll_speed;  // 开始惯性动画
}
```

- 滚轮事件产生 delta 增量，按 `scroll_speed` 系数缩放。
- Delta 的方向与滚动方向匹配（Mac 自然滚动 vs Windows 传统滚动由平台处理）。
- 设置 `scroll_velocity` 启动惯性滑动。

### 拖拽滑块

1. MouseDown 在滑块区域上时，记录 `is_dragging = true` 和 `thumb_start_pos`。
2. MouseMove 时计算拖拽增量，转换为 `scroll_pos` 变化。
3. MouseUp 时停止拖拽，`scroll_velocity` 根据释放速度计算惯性。

### 惯性滑动

```rust
fn apply_inertia(&mut self, dt: f64) {
    if self.scroll_velocity.abs() > 0.01 {
        self.scroll_pos = (self.scroll_pos + self.scroll_velocity * dt)
            .clamp(0.0, self.max_scroll_pos);
        self.scroll_velocity *= (1.0 - friction * dt);  // 摩擦减速
    } else {
        self.scroll_velocity = 0.0;
    }
}
```

- 在 `frame_event` 中每帧调用 `apply_inertia`。
- 摩擦力系数（friction）控制减速快慢，值越大停止越快。
- 当速度小于阈值时归零。

### 键盘导航（scroll_to_key）

通过 `scroll_to_key` 属性支持键盘导航：

```rust
fn scroll_to(&mut self, cx, target: LiveId) {
    // 1. 查找目标 Widget 的位置
    // 2. 计算需要的 scroll_pos 使其可见
    // 3. 使用动画（anim_timeline）平滑滚动到目标位置
}
```

---

## 动画集成

ScrollBar 使用两套动画机制：

1. **Animator** — 用于滑块显隐动画（never show / always show / show on hover）。
2. **惯性动画** — 手动在 `frame_event` 中处理 `scroll_velocity` 的衰减。

两者协作：Animator 控制滑块的透明度过渡，惯性动画控制滚动位置的物理运动。

---

## 方向控制

`orientation` 字段决定是水平还是垂直滚动条：

- `Axis::X`：横向滚动，`scroll_pos` 影响子组件的 x 偏移。
- `Axis::Y`：纵向滚动，`scroll_pos` 影响子组件的 y 偏移。

逻辑完全对称，仅在计算滑块位置和滚轮 delta 时根据方向选取对应坐标轴。
