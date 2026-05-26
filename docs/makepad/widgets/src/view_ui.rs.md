# view_ui.rs — View UI 预设组件

## 文件概述

定义了一系列基于 `View` 的预设 UI 组件，每个都是带有特定样式和功能预设的 `View` 变体：

1. `SolidView` — 纯色背景视图
2. `RoundedView` — 圆角背景视图
3. `ScrollXView` — 水平滚动视图
4. `ScrollYView` — 垂直滚动视图
5. `ScrollXYView` — 双向滚动视图

这些组件通过在 `lib.rs` 中导出自带样式的 View 来实现：

```rust
// lib.rs
pub use view_ui::SolidView;
pub use view_ui::RoundedView;
```

---

## 各组件说明

### SolidView

带纯色背景的 View：

- 继承 `View` 的所有功能（容器、布局、事件路由）。
- 预设 `draw_bg` 的背景色和基础样式。
- 常用于分割面板、工具栏、侧边栏背景。

### RoundedView

带圆角背景的 View：

- 在 SolidView 基础上增加了 `border_radius` 圆角设置。
- 用于卡片、弹窗、按钮等需要圆角边界的容器。

### ScrollXView / ScrollYView / ScrollXYView

滚动容器 View：

- 集成 `ScrollBars` 和 `ScrollBar` 组件管理滚动条。
- `scroll_walk` 字段提供滚动偏移约束。
- 布局计算时考虑滚动位置：
  - 垂直滚动：`scroll_y` 偏移影响子组件的 y 位置
  - 水平滚动：`scroll_x` 偏移影响子组件的 x 位置
  - 双向滚动：同时应用两个方向的偏移
- 滚动区域通过 `overflow: Scroll` 标志启用裁剪。
- 内容超出可视区域时自动显示滚动条。
- 鼠标滚轮事件自动路由到 ScrollBar 的滚动逻辑。

---

## 预设实现方式

这些预设通过 Rust 类型别名 + 脚本 DSL 实现。Rust 层面定义新类型包装 View，脚本层面（`live_design!` / `script_mod!`）为其配置默认样式和子组件：

```
// 概念示例（在脚本 DSL 中）
mod.widgets.SolidView = mod.widgets.View {
    draw_bg: {
        color: #333
    }
}

mod.widgets.ScrollYView = mod.widgets.View {
    flow: Down
    overflow: Scroll
    scroll_y := ScrollBar {}
}
```
