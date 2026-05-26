# adaptive_view.rs — 自适应视图组件

## 概述
`AdaptiveView` 是一个可折叠的多面板视图。它包含多个 `AdaptiveViewPanel`，每个面板有一个标题栏和一个内容区域。面板支持折叠/展开切换。当面板折叠时，标题栏仍然可见，但内容区域收缩到 0 高度。非常适合实现预配置/设置类别面板。

## 核心结构

### AdaptiveView
- **`draw_bg: DrawQuad`**：视图背景绘制器。
- **`panels: Vec<AdaptiveViewPanel>`**：面板列表。

### AdaptiveViewPanel
简单结构，包含 `header_widget: WidgetRef`（标题栏）和 `body_walk: Walk`（内容区域布局参数）。

## 核心方法

### Widget 实现

**`draw_walk`**：遍历所有面板绘制。对每个面板，检查头部是否有 FoldButton——如果 FoldButton 处于折叠状态，则跳过面板内容区域的绘制；如果处于展开状态，则正常绘制。不直接使用 FoldHeader 的动画收缩，而是直接跳过内容绘制以节省渲染成本。

**`handle_event`**：无特殊事件处理，委托给子组件。

### 面板管理

**`panel_count`**：返回面板数量。

**`panel_header_widget`**：按索引返回面板的头部组件引用。

## 与 FoldHeader 的区别

`AdaptiveView` 是更轻量的多面板折叠实现：
- **AdaptiveView**：多个面板，每个面板通过头部中的 FoldButton 控制折叠/展开。折叠时完全跳过内容绘制（不保留动画高度过渡）。
- **FoldHeader**：单个面板，通过动画 `opened: f64` 实现平滑的高度收缩/展开动画过渡。

`AdaptiveView` 适合"设置面板"等场景，多个预设面板并列，每个可以独立折叠，不需要平滑动画。
