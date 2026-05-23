# view.rs — View 容器组件：布局优化、子组件遍历、事件冒泡与动画集成

## 文件概述

`View` 是 Makepad 最核心的容器组件。它负责：
1. 管理子 Widget 列表的布局和绘制
2. 实现三种绘制优化策略（`ViewDraw`）
3. 事件路由与冒泡控制
4. 动画系统集成（Animator 嵌入）
5. 布局约束传递

---

## `View` 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct View {
    #[source] source: ScriptObjectRef,
    #[walk] walk: Walk,
    #[layout] layout: Layout,
    #[redraw] #[live] draw_bg: DrawQuad,
    #[live] animator: Animator,              // 嵌入式动画引擎
    #[live] view_draw: ViewDraw,             // 绘制优化策略
    #[live] capture_when_not_self: bool,     // 事件捕获溢出控制
    #[live] event_capture: EventCaptureMode, // 事件捕获模式
    #[walk] scroll_walk: Option<Walk>,       // 滚动约束（用于 ScrollView）
    #[rust] view_areas: Vec<Area>,           // 子组件 Area 集合
    #[rust] area: Area,
    #[rust] children: Vec<Box<dyn Widget>>,  // 子 Widget 列表
}
```

关键字段：
- `view_draw`：控制绘制优化策略（None/DrawList/Texture）。
- `children`：子 Widget 的动态数组，每个子组件是一个 `Box<dyn Widget>`。
- `animator`：嵌入式 Animator，可以为 View 内外的事件触发动画。
- `view_areas`：收集所有子组件的点击区域，用于事件路由。

---

## ViewDraw 绘制优化策略

```rust
pub enum ViewDraw {
    None,                        // 无优化，直接逐个绘制子组件
    DrawList(Vec<DrawListCall>), // 使用 DrawList 缓存绘制调用
    Texture(DrawTexture),        // 渲染到离屏纹理再整体绘制
}
```

### None 策略

默认模式。`draw_walk` 时直接迭代 `children`，对每个子组件调用 `draw_walk`：

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    cx.begin_turtle(walk, self.layout);
    for child in self.children.iter_mut() {
        let step = child.draw_walk(cx, scope, walk);
        // 处理 step
    }
    cx.end_turtle_with_area(&mut self.area);
    DrawStep::Done(self.area)
}
```

优点：简单、无额外开销。适用于子组件数量较少的视图。

### DrawList 策略

将绘制调用缓存到 `Vec<DrawListCall>` 中：

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    // 1. 检查缓存是否过期
    // 2. 如果有效：直接重放 DrawList 命令
    // 3. 如果无效：重建 DrawList
    // 4. DrawList 中每条命令包含：
    //    - shader_id: 要调用的着色器
    //    - uniform_data: uniform 数据
    //    - vertex_data: 顶点数据
}
```

适用于子组件数量中等且结构相对稳定的视图。通过缓存绘制命令避免重复遍历和属性解析。

### Texture 策略

渲染到离屏纹理：

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    // 1. 创建离屏 DrawTexture
    // 2. 将子组件渲染到纹理
    // 3. 将纹理作为一个整体绘制到屏幕上
    // 4. 子组件的事件通过纹理坐标映射处理
}
```

适用于复杂视图（如 Dock 面板、嵌套布局）。渲染一次后作为纹理整体绘制，减少 draw call 数量。

---

## 子组件遍历：yield 模式

`View` 的 `draw_walk` 使用 yield 模式遍历子组件：

1. 从 `children` 中逐个取出 `Box<dyn Widget>`。
2. 调用 `child.draw_walk()` 获得 `DrawStep`。
3. 将子组件的 `DrawStep` 转换为 `DrawStepTuple` 进行模式匹配。
4. 对 `DrawStepType::Step` 类型的子组件，记录其 `Area` 到 `view_areas`。
5. 如果子组件产生了 `SkipStep`，保存跳过步骤供后续处理。
6. 返回 `DrawStep::Done(self.area)` 或 `DrawStep::Step(self.area)` 给父容器。

---

## 事件路由

```rust
fn handle_event(&mut self, cx, event, scope) {
    // 1. 从 WidgetTree 查询命中目标
    // 2. 如果是子组件的 Area 命中，路由到对应子组件
    // 3. 检查 event_capture 标志：
    //    - EventCaptureMode::Sink → 事件不继续冒泡
    //    - EventCaptureMode::Route → 事件继续向上冒泡
    // 4. 如果设置了 capture_when_not_self，处理捕获溢出
}
```

### EventCaptureMode

```rust
pub enum EventCaptureMode {
    Sink,     // 消费事件，停止冒泡
    Route,    // 事件继续传递
}
```

### 事件查找

`view_areas` 保存了所有子组件的 Area，通过 `cx.widget_event(event, area)` 将事件路由给正确的子组件：

```rust
fn handle_event(&mut self, cx, event, scope) {
    for area in &self.view_areas {
        let capture = cx.widget_event(event, *area);
        if capture == EventCapture::Sink {
            return EventCapture::Sink;
        }
    }
    EventCapture::Route
}
```

---

## 动画集成

Animator 字段允许 View 直接处理动画：

```rust
#[live] animator: Animator,

// Animator 派生宏自动生成 frame_event:
fn frame_event(&mut self, cx, scope) {
    self.animator.animate(cx);
}
```

- `animator.animate(cx)` 检查动画状态变化，如果动画活跃则 `cx.request_redraw()`。
- 动画属性通过脚本 DSL 中 `+:` 语法设置，覆盖 View 子属性。

---

## 布局传播

`draw_walk` 中布局传递流程：

```
Parent View
  └─ walk (约束)
      └─ View.begin_turtle(walk, layout)
          ├─ 计算 content_size
          ├─ 遍历 children:
          │   ├─ 计算每个 child 的 walk 约束
          │   ├─ child.draw_walk(cx, scope, child_walk)
          │   └─ 处理 child 返回的 DrawStep
          └─ end_turtle_with_area
```

- `walk` 约束由父容器提供，包含 `Walk { x, y, width, height, margin, ... }`。
- `layout` 字段控制对齐方式（`Align`）、方向（Flow）、间距（spacing）等布局参数。
- 固定尺寸子组件按指定尺寸绘制，Fill/Hug 子组件根据剩余空间和内容尺寸动态计算。

---

## scroll_walk 与滚动集成

`scroll_walk: Option<Walk>` 用于 ScrollView 环境：

- 当 View 作为 ScrollView 内容时，scroll_walk 提供滚动的偏移约束。
- 不为 `None` 时，子组件的绘制坐标会根据 scroll_walk 的偏移量调整。
- 实际滚动由 ScrollBar/ScrollBars 组件驱动，View 仅负责应用偏移。
