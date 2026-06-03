# tab_checkbox.rs

CheckBox、Toggle 及自定义复选框展示页面。

```rust
mod.widgets.DemoCheckBox = UIZooTabLayout_B{
    desc +: { Markdown{body: "# CheckBox\n\nCheckboxes allow toggling options on/off."} }
```

### CheckBox 系列

```rust
CheckBox{text: "CheckBox"}
CheckBox{ animator +: { disabled: { default: @on } } }
CheckBoxFlat{text: "CheckBoxFlat"}
```

- 标准 CheckBox。
- 禁用的 CheckBox。
- `CheckBoxFlat`: 扁平样式复选框。

### Toggle 系列

```rust
Toggle{text: "Toggle"}
ToggleFlat{text: "ToggleFlat"}
```

开关控件，可替代复选框的视觉方案。

### 输出 Demo

```rust
simplecheckbox := CheckBox{text: "CheckBox"}
simplecheckbox_output := Label{text: ""}
```

在 app.rs 中处理其 `changed` 事件：
```rust
if let Some(check) = self.ui.check_box(cx, ids!(simplecheckbox)).changed(actions) {
    let lbl = self.ui.label(cx, ids!(simplecheckbox_output));
    lbl.set_text(cx, &format!("{} {}", self.counter, check));
}
```
每次勾选切换时更新输出 Label，显示计数器和当前状态。

### 自定义 CheckBox

```rust
CheckBoxCustom{
    text: "CheckBoxCustom"
    align: Align{x: 0. y: 0.5}
    padding: Inset{top: 0. left: 0. bottom: 0. right: 0.}
    margin: Inset{top: 0. left: 0. bottom: 0. right: 0.}
    label_walk: Walk{ width: Fit height: Fit margin: theme.mspace_h_1{left: 5.5} }
    draw_icon +: { svg: crate_resource("self:resources/Icon_Favorite.svg") }
    icon_walk: Walk{ width: 13.0 height: Fit }
}
```

- 使用自定义 SVG 图标代替默认勾选标记。
- 通过 `label_walk` 和 `icon_walk` 精确控制布局。
- `CheckBoxCustom` 是 Makepad 内置变体。
