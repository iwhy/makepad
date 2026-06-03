# tab_slider.rs

Slider 滑块展示页面 — 演示多种 Slider 变体和参数配置。

## Slider（标准）

```rust
Slider{text: "Default"}
Slider{text: "Default, disabled" animator +: { disabled: { default: @on } } }
Slider{text: "min/max" min: 0. max: 100.}
Slider{text: "precision" precision: 20}
Slider{text: "stepped" step: 0.1}
```

参数说明：
- `min` / `max`: 值范围（默认 0.0 ~ 1.0）。
- `precision`: 精度（值被分为多少份，如 precision: 20 表示 0.05 步进）。
- `step`: 固定步进值。

## SliderGradientY

```rust
SliderGradientY{text: "Default"}
SliderGradientY{text: "min/max" min: 0. max: 100.}
SliderGradientY{text: "precision" precision: 20}
SliderGradientY{text: "stepped" step: 0.1}
```

垂直渐变样式的滑块。

## SliderGradientX

```rust
SliderGradientX{text: "Default"}  // 水平渐变样式
```

## SliderFlat

```rust
SliderFlat{text: "Default"}  // 扁平样式
```

## SliderMinimal

```rust
SliderMinimal{text: "Default"}  // 极简样式
```

## SliderMinimalFlat

```rust
SliderMinimalFlat{text: "Default"}  // 极简扁平样式
```

## SliderRound

```rust
SliderRound{text: "Default"}
```

圆形样式的滑块，支持完整的颜色自定义：

```rust
SliderRound{
    draw_bg +: {
        val_color: uniform(#xF08)  val_color_hover: uniform(#xF4A)
        val_color_focus: uniform(#xC04)  val_color_drag: uniform(#xF08)
        val_color_2: uniform(#xF08) ...
        handle_color: uniform(#xF)  handle_color_hover: uniform(#xF) ...
    }
}
// 全自定义颜色方案
```

还支持 `label_size: 150.` 参数控制标签宽度。

## SliderRoundGradientY / SliderRoundGradientX / SliderRoundFlat

```rust
SliderRoundGradientY{text: "min/max" min: 0. max: 100.}
SliderRoundGradientX{text: "min/max" min: 0. max: 100.}
SliderRoundFlat{text: "min/max" min: 0. max: 100.}
```

不同视觉风格的圆形滑块变体，共 8 种 Slider 类型展示。
