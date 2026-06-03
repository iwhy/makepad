# tab_icon.rs

Icon SVG 矢量图标展示页面。

```rust
mod.widgets.DemoIcon = UIZooTabLayout_B{
    desc +: { Markdown{body: "# Icon\n\nIcons display SVG vector graphics."} }
```

### Standard Icon

```rust
Icon{ draw_icon +: {svg: crate_resource("self:resources/Icon_Favorite.svg")} }
```

基本 SVG 图标，使用 `crate_resource` 引用资源文件。

### Gradient 系列

```rust
IconGradientX{ icon_walk: Walk{width: 100.} draw_icon +: {svg: crate_resource("self:...")} }
IconGradientY{ icon_walk: Walk{width: 100.} draw_icon +: {svg: crate_resource("self:...")} }
```

- `IconGradientX`: 水平渐变色图标。
- `IconGradientY`: 垂直渐变色图标。

### 样式参考

```rust
Icon{
    width: Fit  height: Fit
    icon_walk: Walk{ width: 50. margin: 10. }
    draw_bg +: {color: uniform(#f00)}
    draw_icon +: {
        svg: crate_resource("self:resources/Icon_Favorite.svg")
        color: #f0f
        color_2: #ff0
    }
}
```

- `draw_bg`: 添加红色背景。
- `draw_icon.color`: 图标颜色（#f0f 紫色），`color_2`（#ff0 黄色）用于渐变。
- `icon_walk`: 控制图标尺寸和边距。
