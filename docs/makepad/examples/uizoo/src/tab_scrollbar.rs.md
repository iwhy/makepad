# tab_scrollbar.rs

ScrollBar 滚动条展示页面。

```rust
mod.widgets.DemoScrollBar = UIZooTabLayout_B{
    desc +: { Markdown{body: "# ScrollBar\n\nScrollBars enable scrolling through content."} }
    demos +: {
        GradientYView{
            height: 4000.  width: Fill
            draw_bg +: { color_2: uniform(#f00) }
        }
        scroll_bars: ScrollBars{
            scroll_bar_x.drag_scrolling: true
            scroll_bar_y.drag_scrolling: true
        }
    }
}
```

- `GradientYView`: 垂直渐变视图（从默认颜色渐变到红色），高度 4000px 以产生滚动需求。
- `ScrollBars`: 同时启用水平和垂直滚动条，均支持拖拽滚动（`drag_scrolling: true`）。
- 关键点：`scroll_bars` 属性直接附加在 `UIZooTabLayout_B` 的 `demos` View 上，使得 Demo 内容区获得滚能力。
