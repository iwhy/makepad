# tab_slug.rs

SLUG（Signed Distance Field Large Upright Glyph）文本渲染路径测试页面。

## 背景

SLUG 是 Makepad 用于大字号文本的渲染路径。在 Linux 上，大于一定尺寸的文字会从常规的 raster/MSDF 路径切换到 SLUG 路径。此 Demo 用于对比两种路径的渲染效果。

## 辅助模板

```rust
let SlugDemoCard = RoundedView{ ... }      // 对比卡片容器
let SlugProbeGrid = View{ flow: Flow.Right{wrap: true} ... }  // 诊断网格
let SlugLargeDefaultLabel = Label{ draw_text.text_style.font_size: 160. }  // 大字号标签
```

## 对比测试

### Plain Label

```rust
Label{font_size: 32. text: "Ag"}  // Below Linux SLUG cutoff
Label{font_size: 192. text: "Ag"} // Above Linux SLUG cutoff
```

相同文本、不同字号，对比 SLUG 启用前后的渲染效果。

### Gradient Label

```rust
LabelGradientX{font_size: 32. text: "SLUG"}  // 渐变文字（小）
LabelGradientX{font_size: 192. text: "SL"}   // 渐变文字（大）
```

### Custom Text Shader

```rust
Label{
    font_size: 32.
    get_color: fn() -> vec4 { return mix(theme.color_makepad #0000 self.pos.x) }
    text: "WAVE"
}
Label{
    font_size: 192.
    get_color: fn() -> vec4 { return mix(theme.color_makepad #0000 self.pos.x) }
    text: "W"
}
```

自定义像素着色器在两种渲染路径下的对比。

### LinkLabel

```rust
LinkLabel{font_size: 28. text: "Open docs"}
LinkLabel{font_size: 144. text: "Go"}
```

可交互链接在 SLUG 路径下的表现。

## 诊断矩阵

### Color Source Probes（颜色来源探测）

```rust
let SlugLargeLiteralLabel = Label{color: #fff font_size: 160.}
let SlugLargeThemeLabel = Label{color: theme.color_text font_size: 160.}
let SlugLargeAccentLabel = Label{color: theme.color_makepad font_size: 160.}
let SlugLargeGradientLabel = Label{color: #x6CF color_2: #xFD6 font_size: 160.}
```

对比不同颜色来源（默认/字面量/主题色/主题强调色/渐变）在 SLUG 路径下的渲染。

### Glyph Shape Probes（字形探测）

```rust
SlugLargeLiteralLabel{text: "A"}  // 大写含内腔
SlugLargeLiteralLabel{text: "g"}  // 小写下伸
SlugLargeLiteralLabel{text: "W"}  // 宽字母
SlugLargeLiteralLabel{text: "S"}  // 曲线
SlugLargeLiteralLabel{text: "L"}  // 简单角
SlugLargeLiteralLabel{text: "O"}  // 闭环
SlugLargeLiteralLabel{text: "8"}  // 双内腔
SlugLargeLiteralLabel{text: "y"}  // 下伸
```

独立测试不同字形几何形状在 SLUG 路径下是否正常。

### Custom Shader Probes

```rust
SlugLargeCustomLiteralLabel{text: "Ag"}  // 自定义着色器 + 字面量颜色
SlugLargeCustomThemeLabel{text: "Ag"}    // 自定义着色器 + 主题色
SlugLargeCustomLiteralLabel{text: "W"}   // 自定义着色器 + 字面量颜色 + W
SlugLargeCustomThemeLabel{text: "W"}     // 自定义着色器 + 主题色 + W
```

组合测试自定义着色器 + 颜色来源 + 字形。
