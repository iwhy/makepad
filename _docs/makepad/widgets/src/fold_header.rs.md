# fold_header.rs — 折叠面板头部

## 概述
`FoldHeader` 是一个可折叠/展开的面板容器，包含头部区域（header）和主体区域（body）。支持平滑的展开/折叠动画，主体部分根据 `opened` 属性动画改变高度。

## 核心结构

### FoldHeader
- **`header: WidgetRef`**：头部区域（通常包含 FoldButton 和标题）。
- **`body: WidgetRef`**：主体区域（折叠时隐藏）。
- **`opened: f64`**：展开程度（0.0=完全折叠，1.0=完全展开）。
- **`body_walk: Walk`**：主体区域的布局参数。
- **`rect_size: f64`**：主体内容的实际高度缓存。

### DrawState
两阶段绘制状态：`DrawHeader`（绘制头部和准备主体容器）和 `DrawBody`（绘制主体内容）。

## 核心方法

### Widget 实现

**`handle_event`**：处理动画器事件触发重绘，委托事件给 header 和 body 的子组件。从 `Event::Actions` 中监听 header 内 FoldButton 的 `Opening`/`Closing` 动作，同步播放 `active.on`/`active.off` 动画。

**`draw_walk`**：两阶段绘制。第一阶段绘制 header 和主体测量容器；第二阶段绘制 body 内容。首次渲染（`rect_size == 0`）使用原始 body_walk 测量实际内容高度。后续渲染根据 `self.rect_size * self.opened` 计算动画高度，并使用 `scroll` 实现平滑收缩效果。

### 状态控制

**`set_is_open`**：通过动画器切换展开/折叠状态，同步更新 header 中的 FoldButton（如果存在）。

**`is_open`**：检测当前是否处于展开状态。
