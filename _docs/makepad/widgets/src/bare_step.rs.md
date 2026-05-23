# bare_step.rs — 空 DrawStep 占位 widget

## 整体职能
`BareStep` 是一个极简的占位 widget，其 `draw_walk` 方法仅返回 `DrawStep::done()`，不产生任何绘制。它用于需要 widget 类型占位但不需要实际渲染的场景，例如作为条件性内容的占位符、或者作为 WidgetRef 的默认值。

## 主要数据结构
- **`BareStep`**：极简结构体，仅包含 `#[walk] walk: Walk` 和 `#[layout] layout: Layout` 两个必需的布局字段，以及一个空的 `#[rust]` 字段用于运行时标记。

## 方法与实现逻辑

### `fn script_component` — 脚本注册
以最简形式将 `BareStep` 注册到脚本运行时，不添加任何默认样式或子组件。

### `fn draw_walk` — 绘制
立即返回 `DrawStep::done()`，不占用任何绘制周期和 Turtle 上下文。父 widget 的布局流程仍会为其分配空间，但完全不产生绘制内容。

### `fn handle_event` — 事件处理
空实现，不处理任何事件。所有输入事件透传给父 widget。
