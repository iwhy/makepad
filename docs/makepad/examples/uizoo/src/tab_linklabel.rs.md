# tab_linklabel.rs

LinkLabel 可点击文本链接展示页面。

```rust
mod.widgets.DemoLinkLabel = UIZooTabLayout_B{
    desc +: { Markdown{body: "# LinkLabel\n\nLinkLabels are clickable text links."} }
```

### 标准链接

```rust
LinkLabel{text: "Click me!"}
```

基本可点击文本链接。

### Disabled

```rust
LinkLabel{
    text: "Click me!"
    animator +: { disabled: { default: @on } }
}
```

禁用状态下的链接标签。

### 样式参考

```rust
LinkLabel{
    draw_text +: {
        color: #xA  color_hover: #xC  color_down: #8
        text_style +: { font_size: 20. line_spacing: 1.4 }
    }
    draw_bg +: {
        color: uniform(#x0A0)  color_hover: uniform(#x0C0)  color_down: uniform(#080)
    }
    icon_walk: Walk{ width: 20. height: Fit }
    draw_icon +: {
        color: #xA00  color_hover: #xC00  color_down: #800
        svg: crate_resource("self:resources/Icon_Favorite.svg")
    }
    text: "Click me!"
}
```

完整自定义样式：
- `draw_text`: 文字颜色三态（normal/hover/down）。
- `draw_bg`: 背景颜色三态。
- `draw_icon`: SVG 图标 + 颜色三态。
- `icon_walk`: 图标尺寸控制。
