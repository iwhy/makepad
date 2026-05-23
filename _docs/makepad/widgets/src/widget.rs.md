# widget.rs — Widget Trait、DrawStep 状态机、WidgetRef/WidgetSet 智能指针

## 文件概述

这是 Makepad widgets crate 最核心的文件。定义了整个 UI 框架的基石：

1. `DrawStep` — 绘制过程的 yield 状态机
2. `Widget` trait — 每个 UI 组件必须实现的接口
3. `WidgetNode` — Widget 树节点枚举（Widget / WidgetSet / WidgetArray）
4. `WidgetRef` / `WidgetSet` — 脚本化 Widget 的智能指针和集合
5. `EventCapture` — 事件冒泡/捕获阶段控制
6. `CxWidgetExt` — 为 `Cx` 上下文提供的 Widget 树扩展方法

---

## `DrawStep` 枚举

绘制过程的 yield 状态机，替代传统的直接返回。每个 `draw_walk` 调用通过 yield 模式让父容器控制迭代流程：

```rust
pub enum DrawStep {
    Done(Area),                        // 绘制完成，返回点击区域
    Step(Area),                        // 产生一个子区域，需要父容器继续迭代
    SkipStep(Area, Vec<DrawStep>),     // 跳过当前子区域，稍后处理
    StepWidgetNodeMut(WidgetNodeMut),  // 动态 Widget 节点需要绘制
    SkipWidgetNodeMut(WidgetNodeMut, Vec<DrawStep>), // 动态节点稍后处理
    SkipWidgetCapture(Area, WidgetCapture), // 捕获的 Widget 事件处理
}
```

关键设计意图：
- `Done(Area)` — 完成绘制，提供点击命中的 Area。正常流程的终点。
- `Step(Area)` — 容器（如 View）在绘制其子组件时，每绘制一个子组件 yield 一个 Step，父容器据此继续下一个子组件的绘制。
- `SkipStep` — prefabs 系统中，当某个子组件被跳过时保存剩余的 draw steps。
- `StepWidgetNodeMut` — 用于 `WidgetNode::WidgetSet` 和 `WidgetNode::WidgetArray` 类型的动态子元素。当 Widget 自身持有子 Widget 列表时，需要通过这种方式让父容器能访问并绘制它们。
- `SkipWidgetNodeMut` — 对应的跳过版本。
- `SkipWidgetCapture` — 当一个 Widget 捕获了事件（如拖拽），后续的事件直接路由给捕获者，不再经过正常事件路径。

每个 Step 都携带 `Area`，用于点击命中检测。`DrawStep` 实现了 `is_step()` / `step()` 等辅助方法，配合 `while let` 循环实现迭代。

---

## `Widget` trait

### `draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep`

框架最核心的绘制方法：

1. 接收父容器提供的 `walk` 布局约束。
2. 内部通过 `cx.begin_turtle(walk, self.layout)` 建立布局上下文。
3. 根据自身类型执行具体绘制逻辑：
   - 叶子 Widget（Label、Button）：直接调用 `draw_quad` / `draw_text` 后通过 `cx.end_turtle_with_area` 结束。
   - 容器 Widget（View）：迭代子组件，对每个子组件调用 `child.draw_walk()`，每完成一个 yield `DrawStep::Step`。
4. 返回 `DrawStep::Done(area)` 或 `DrawStep::Step(area)` 让父容器继续。

### `handle_event(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope)`

事件处理方法：

1. 通过 `self.match_event(cx, event)` 分发到具体 handler（如 `clicked()`、`hovered()`）。
2. 将事件传递给子组件：通过 `cx.widget_event(event, area)` 或 `self.view.handle_event()` 路由。
3. 事件冒泡顺序：子组件优先，父组件在后。
4. 对于捕获的事件，直接由捕获者处理，不经过正常路由。

### `dependencies(&self) -> Option<&[Dependency]>`

动画依赖声明。返回此 Widget 所依赖的动画属性列表，用于 AnimatedWidget 的变更追踪。`None` 表示无依赖。

### `frame_event(&mut self, cx: &mut Cx, scope: &mut Scope)`

帧事件回调。每帧被调用一次，通常用于动画更新。默认空实现。

### `visibility_changed_event(&mut self, cx: &mut Cx, scope: &mut Scope)`

Widget 可见性变化事件。当 Widget 进入/离开可见范围时触发（例如 ScrollView 中的懒加载项），默认空实现。

### `on_startup(&mut self, cx: &mut Cx, scope: &mut Scope)`

启动回调。在 App 首次运行时调用，用于初始化。Root 组件在该方法中触发组件映射注册。

---

## `WidgetNode` 枚举

Widget 树的节点类型：

```rust
pub enum WidgetNode {
    Widget(Box<dyn Widget>),         // 单个 Widget
    WidgetSet(Box<dyn WidgetSet>),   // 具名 Widget 集合
    WidgetArray(Vec<WidgetNode>),    // 动态 Widget 数组
}
```

- `Widget`：持有 `Box<dyn Widget>` trait 对象，是基本的叶子或容器。
- `WidgetSet`：具名 Widget 集合，通常对应 DSL 中的 `mod.widgets.X` 定义。包含一个 `HashMap<LiveId, WidgetNode>` 映射。
- `WidgetArray`：动态 Widget 数组，用于列表等可变数量子组件。draw_walk 时转换为 `WidgetArrayIter` 进行遍历。

---

## `WidgetRef` 和 `WidgetSet`

### `WidgetRef`

脚本化的智能指针，内部持有 `ScriptHandleRef`：

```rust
pub struct WidgetRef {
    inner: WidgetRefInner,  // Direct / Indirect
    ids: Option<Ids>,
}
```

- `Direct`：直接持有 `ScriptHandleRef`（UI 树中的引用）。
- `Indirect`：通过 `WidgetSet` 的 id 映射间接引用。
- 提供便捷方法如 `.button(id).clicked(actions)`、`.view(id).set_text(cx, text)` 等。

关键方法：
- `from_resource()` / `from_resource_with_id()`：从资源加载 Widget 子树。
- `button()` / `view()` / `label()` 等：通过 `id!` 宏获取子组件引用。
- `draw_walk()` / `handle_event()`：代理到内部的 WidgetNode。

### `WidgetSet`

`WidgetSet` trait 定义了具名 Widget 集合：

```rust
pub trait WidgetSet {
    fn widget_ref(&self, id: LiveId) -> WidgetNode;  // 通过 LiveId 获取子 Widget
    fn widget_ids(&self) -> &SetId;  // 返回该集合包含的所有子 Widget id
}
```

`SetId` 结构维护 `Vec<LiveId>`，用于快速查找和绘制。脚本中用 `:=` 语法定义的命名组件会触发 `SetId` 的注册。

`WidgetSet` 体系下还有：
- `WidgetArrayWalk`：处理 `WidgetArray` 类型的动态列表绘制。
- `WidgetCapture`：事件捕获状态管理，用于拖拽等操作。

---

## `EventCapture`

控制事件在 Widget 树中的路由：

```rust
pub enum EventCapture {
    Sink,       // 事件被捕获并消费，不再冒泡
    Route,      // 事件继续正常路由
}
```

由 `handle_event_with_capture` 返回：
- `Sink`：完全消费该事件（如 Button 的 click 事件，防止传递给下面的 Widget）。
- `Route`：事件可以继续传递。

---

## Turtle/布局系统集成

Widget 的绘制离不开 turtle 系统：

- `cx.begin_turtle(walk, layout)` — 根据 walk 约束和 layout 参数创建一个绘图区域。
- `cx.turtle().rect()` — 获取当前 turtle 相关的矩形区域。
- `cx.end_turtle_with_area(area)` — 结束 turtle，记录点击区域。
- `cx.turtle().add_named(area, id)` — 将某个子区域注册为具名区域（用于事件路由）。

---

## CxWidgetExt

为 `Cx` 上下文添加的扩展方法：

```rust
pub trait CxWidgetExt {
    fn widget_event(&mut self, event: &Event, area: Area) -> EventCapture;
}
```

- `widget_event(event, area)` — 将事件路由到指定 Area 的 Widget，处理命中测试和事件分发。
- 内部通过 `cx.widget_capture()` 获取事件捕获状态，决定是否跳过正常路由。
