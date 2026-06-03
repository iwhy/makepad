# tab_rotary.rs

Rotary 旋钮控件展示页面 — 演示圆形拨盘选择器的变体和参数。

## Rotary 系列

```rust
Rotary{text: "Label"}

Rotary{ draw_bg +: { val_size: uniform(10.) val_padding: uniform(2.) gap: uniform(0.) } }
// 紧凑布局：大值点、小间隙、无间隔

Rotary{ draw_bg +: { val_size: uniform(5.) val_padding: uniform(2.5) gap: uniform(180.) } }
// 半圆布局：小值点、有间隔

Rotary{ draw_bg +: { val_size: uniform(5.) val_padding: uniform(0.) gap: uniform(180.) }
        animator +: { disabled: { default: @on } } }
// 禁用状态

Rotary{ width: Fill height: 150
        draw_bg +: { val_size: uniform(10.) val_padding: uniform(5.) } }
// Fill 宽度，更大尺寸
```

Rotary 参数：
- `val_size`: 值指示点的大小。
- `val_padding`: 值点之间的间距。
- `gap`: 旋钮弧线的间隙角度（0° = 完整圆环，180° = 半圆）。

## RotaryGradientY

```rust
RotaryGradientY{text: "Label"}
RotaryGradientY{ draw_bg +: {gap: uniform(0.)} }
RotaryGradientY{ draw_bg +: {gap: uniform(180.)} }
RotaryGradientY{ animator +: { disabled: { default: @on } } draw_bg +: {val_size: uniform(20.)} }
RotaryGradientY{ width: Fill height: 150 draw_bg +: {val_size: uniform(10.) val_padding: uniform(5.)} }
```

垂直渐变样式的旋钮变体。

## RotaryFlat

```rust
RotaryFlat{text: "Label"}
RotaryFlat{ draw_bg +: {gap: uniform(0.)} }
RotaryFlat{ draw_bg +: {gap: uniform(180.)} }
RotaryFlat{ animator +: { disabled: { default: @on } } draw_bg +: {val_size: uniform(10.)} }
RotaryFlat{ width: Fill height: 150 draw_bg +: {val_size: uniform(10.) val_padding: uniform(8.)} }
```

扁平样式的旋钮变体。
