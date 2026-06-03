# layout_templates.rs

布局模板定义 — 提供可复用的页面布局和行容器。

```rust
mod.widgets.UIZooTabLayout_B = View{
    height: Fill width: Fill
    flow: Right
    padding: 0  spacing: 0.
```

`UIZooTabLayout_B` 是所有 Demo Tab 的基础布局模板，水平分为两栏：

### 左侧描述栏 (desc)

```rust
desc := RoundedView{
    width: 350. height: Fill
    show_bg: true
    draw_bg +: { color: theme.color_inset border_radius: uniform(theme.corner_radius) }
    padding: theme.mspace_3{top: 0. right: theme.space_2}
    margin: theme.mspace_v_2
    flow: Down  spacing: theme.space_2
    scroll_bars: ScrollBars{ show_scroll_y: true scroll_bar_y.drag_scrolling: true }
}
```

- 固定宽度 350px，左侧放置各 Widget 的说明文档（Markdown）。
- 圆角背景、内边距、垂直滚动。

### 右侧 Demo 区域 (demos)

```rust
demos := View{
    width: Fill height: Fill
    flow: Down  spacing: theme.space_2
    padding: theme.mspace_3{right: (theme.space_2 * 3.0)}
    margin: theme.mspace_v_2
    scroll_bars: ScrollBars{ show_scroll_y: true scroll_bar_y.drag_scrolling: true }
}
```

- `Fill` 宽度填充剩余空间。
- 垂直排列 Demo 内容，支持滚动。

### UIZooRowH

```rust
mod.widgets.UIZooRowH = View{
    height: Fit width: Fill
    spacing: theme.space_2  flow: Right
    align: Align{x: 0. y: 0.5}
}
```

水平行容器，用于在一行内展示多个相关控件。`align: {x: 0, y: 0.5}` 实现左对齐垂直居中。
