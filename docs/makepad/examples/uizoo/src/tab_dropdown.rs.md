# tab_dropdown.rs

DropDown 下拉选择器展示页面。

```rust
mod.widgets.DemoDropdown = UIZooTabLayout_B{
    desc +: { Markdown{body: "# DropDown\n\nDropdowns allow selecting from a list of options."} }
```

### DropDown 系列

```rust
dropdown := DropDown{ labels: ["Value One" "Value Two" "Third" "Fourth Value" "Option E" "Hexagons"] }
dropdown_disabled := DropDown{ labels: [...] animator +: { disabled: { default: @on } } }
```

- 标准 `DropDown`，通过 `labels` 属性设置选项列表。
- 禁用变体。

### 样式变体

```rust
dropdown_flat := DropDownFlat{ labels: [...] }
dropdown_gradient_x := DropDownGradientX{ labels: [...] }
dropdown_gradient_y := DropDownGradientY{ labels: [...] }
```

四种样式变体展示 DropDown 在 Makepad 中的多样化外观：
- `DropDownFlat`: 扁平风格。
- `DropDownGradientX`: 水平渐变。
- `DropDownGradientY`: 垂直渐变。

所有变体共享相同的选项数据（6 个字符串），通过 `labels: [...]` 数组语法配置。
