# tab_label.rs

Label 文本展示页面 — 类型系统和文字截断功能完整演示。

## 基础 Label

```rust
Label{text: "Default single line text"}
```

## Gradient 系列

```rust
LabelGradientX{text: "LabelGradientX"}
LabelGradientX{ draw_text +: { color: #0ff text_style +: { font_size: 20 } } text: "LabelGradientX" }
LabelGradientY{text: "LabelGradientY"}
```

水平/垂直渐变标签变体。

## TextBox

```rust
TextBox{ text: "Sed ut perspiciatis unde omnis iste natus error sit voluptatem..." }
```

多行文本自动换行容器。

## Typographic System（排版系统）

```rust
H1{text: "H1 headline"}    H1italic{text: "H1 italic headline"}
H2{text: "H2 headline"}    H2italic{text: "H2 italic headline"}
H3{text: "H3 headline"}    H3italic{text: "H3 italic headline"}
H4{text: "H4 headline"}    H4italic{text: "H4 italic headline"}
P{text: "P copy text"}     Pitalic{text: "P italic copy text"}
Pbold{text: "P bold copy text"}
Pbolditalic{text: "P bold italic copy text"}
```

展示 Makepad 的完整排版预设系统，包含 H1~H4 标题和正文的 Regular/Italic/Bold/BoldItalic 变体。

## 样式参考

```rust
Label{
    draw_text +: { color: #0ff text_style +: { font_size: 20. line_spacing: 1.4 } }
    text: "You can style text using colors and fonts"
}
```

## Ellipsis 截断演示

展示了多种截断场景：

### 单行截断

```rust
Label{ max_lines: 1 text_overflow: Ellipsis text: "This is a very long label text that should be truncated..." }
```

### 2 行截断

```rust
Label{ max_lines: 2 text_overflow: Ellipsis text: "This is a longer piece of text..." }
```

### Fill 宽度 + 3 行截断（TextBox）

```rust
TextBox{ max_lines: 3 text_overflow: Ellipsis text: "Sed ut perspiciatis..." }
```

### Emoji 截断

```rust
Label{ max_lines: 1 text_overflow: Ellipsis text: "Stars ⭐⭐⭐ and rockets 🚀🚀🚀..." }
```

### CJK 截断

```rust
Label{ width: Fit{max: FitBound.Rel{base: Base.Full, factor: 0.6}} max_lines: 1
       text_overflow: Ellipsis text: "文字のテストです。日本語の文章が長すぎると省..." }
```

### 混合文字截断

```rust
Label{ text: "Hello Привет 👋 world! Multi-script text..." max_lines: 2 }
```

### 纯 Emoji 截断

```rust
Label{ text: "😀😁😂😃😄😅😆😇😈😉😊😋😌😍😎😏..." max_lines: 1 }
```

### FitBound 截断

```rust
Label{ width: Fit{max: FitBound.Rel{base: Base.Full, factor: 0.5}} text_overflow: Ellipsis }
```

`Fit` 宽度 + 相对最大边界（父容器 50%），到达边界后截断。

## 自定义 Shader

```rust
Label{
    draw_text +: {
        get_color: fn() -> vec4 {
            return mix(theme.color_makepad #0000 self.pos.x)
        }
        color: theme.color_makepad
        text_style +: { font_size: 40. }
    }
    text: "OR EVEN SOME PIXELSHADERS"
}
```

自定义像素着色器：`get_color` 根据水平位置从 `theme.color_makepad` 渐变到透明。
