# tab_textinput.rs

TextInput 文本输入展示页面 — 完整演示 TextInput 的各种模式。

## 基础 TextInput

```rust
simpletextinput := TextInput{}
simpletextinput_outputbox := P{text: "Output"}
```

在 app.rs 中处理 `changed` 事件，实时显示输入内容。

## 禁用 / 内联标签 / 预填内容

```rust
TextInput{empty_text: "Inline Label" animator +: { disabled: { default: @on } } }
TextInput{empty_text: "Inline Label"}
TextInput{empty_text: "Some text"}
```

## 样式变体

```rust
TextInputFlat{empty_text: "Inline Label"}
TextInputGradientX{empty_text: "Inline Label"}
TextInputGradientY{empty_text: "Inline Label"}
```

扁平、水平渐变、垂直渐变三种样式。

## 单行 + 水平滚动

```rust
TextInput{
    text: "This is a long piece of text that should overflow..."
    width: Fill
}
TextInput{ width: 200.0 }
TextInput{ width: Fit }  // 在 300px 容器内
```

演示三种宽度模式下文字溢出时的水平滚动：
- `Fill`: 填充满，溢出水平滚动。
- `Fixed 200px`: 固定宽度溢出滚动。
- `Fit`: 自适应宽度，被父容器限制后滚动。

## 多行模式

```rust
multiline_textinput := TextInput{
    is_multiline: true  height: 150.0  width: Fill
}
```

固定高度多行输入，溢出时使用鼠标滚轮或拖拽滚动条。

### 预填内容

```rust
multiline_prefilled := TextInput{
    is_multiline: true  height: 200.0  text: "Line 1: ...\nLine 2: ..."  // 10 lines
}
```

### 只读模式

```rust
multiline_readonly := TextInput{
    is_multiline: true  is_read_only: true  height: 120.0
    text: "This is a read-only..."
}
```

可滚动查看和选择文本，但不可编辑。

### Fit 高度 + 上限

```rust
TextInput{
    is_multiline: true
    height: Fit{max: FitBound.Abs(100)}  // 绝对上限 100px
}
TextInput{
    is_multiline: true
    height: Fit{max: FitBound.Rel{base: Base.Full, factor: 0.3}}  // 相对父容器 30%
}
```

`Fit` + `FitBound` 实现内容自适应高度 + 最大高度限制，超出后滚动。

### 模式切换

```rust
multiline_toggle := CheckBox{text: "Multiline"}
multiline_toggleable := TextInput{text: "Toggle me..." height: Fit width: Fill}
```

在 app.rs 中响应切换：
```rust
if let Some(is_multiline) = self.ui.check_box(cx, ids!(multiline_toggle)).changed(actions) {
    let ti = self.ui.text_input(cx, ids!(multiline_toggleable));
    ti.set_is_multiline(cx, is_multiline);
}
```
勾选时切换到多行模式，取消勾选回到单行模式。
