# tab_button.rs

Button Widget 完整展示页面。

```rust
mod.widgets.DemoButton = UIZooTabLayout_B{
    desc +: { Markdown{body: "# Button\n\nButtons trigger actions when clicked."} }
```

描述栏显示 Button 的简要说明。

### Standard 按钮

```rust
Button{}
Button{ draw_bg +: { color_2: uniform(#f00) ... } }
basicbutton := Button{}
iconbutton := Button{ draw_icon +: { svg: crate_resource("self:resources/Icon_Favorite.svg") } text: "Button" }
```

- 默认按钮。
- 自定义颜色按钮（`color_2` 和 `border_color_2` 系设为红色）。
- `basicbutton`: 带 ID 的按钮，事件在 app.rs 中处理，点击时更新文字。
- `iconbutton`: 带 SVG 图标的按钮。

### 禁用按钮

```rust
Button{ animator +: { disabled: { default: @on } } }
```

通过 `animator.disabled.default: @on` 设置按钮为初始禁用状态。

### ButtonIcon

```rust
ButtonIcon{ draw_icon +: { svg: crate_resource("self:resources/Icon_Favorite.svg") } }
```

纯图标按钮（无文字），使用渐变色。

### Gradient 系列

- **ButtonGradientX**: 水平渐变的按钮，支持自定义 `color`/`color_2`、`border_color`/`border_color_2` 及 hover/down/focus 各态。
- **ButtonGradientXIcon**: 水平渐变图标按钮。
- **ButtonGradientY**: 垂直渐变按钮。
- **ButtonGradientYIcon**: 垂直渐变图标按钮。

所有渐变按钮展示在带自定义颜色的样式中。

### Flat 系列

- **ButtonFlat**: 扁平按钮，可同时显示图标和文字。
  - 变体：`flow: Down` 垂直排布、自定义 `icon_walk`。
- **ButtonFlatIcon**: 纯图标扁平按钮。

### Flatter 系列

- **ButtonFlatter**: 更扁平的按钮（通常无背景）。
- **ButtonFlatterIcon**: 纯图标更扁平按钮。

所有按钮变体演示了 `draw_bg`/`draw_icon`/`draw_text` 的自定义方式和 Makepad 按钮系统的层次结构。
