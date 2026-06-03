# tab_image.rs

Image 位图图片展示页面 — 演示不同 `ImageFit` 模式的效果。

ImageFit 模式包括：

### 默认（Original）

```rust
Image{src: crate_resource("self:resources/ducky.png")}
```

默认按原始尺寸显示图片。

### fit: Stretch

```rust
Image{width: Fill height: Fill src: crate_resource("self:resources/ducky.png") fit: ImageFit.Stretch}
```

拉伸图片以填满容器，会改变宽高比。

### fit: Horizontal

```rust
Image{fit: ImageFit.Horizontal}
```

水平适应：宽度填满容器，高度等比例缩放。

### fit: Vertical

```rust
Image{fit: ImageFit.Vertical}
```

垂直适应：高度填满容器，宽度等比例缩放。

### fit: Smallest

```rust
Image{fit: ImageFit.Smallest}
```

按最小边适配，等比缩放直到一边到达容器边界。

### fit: Biggest

```rust
Image{fit: ImageFit.Biggest}
```

按最大边适配，等比缩放直到一边超出容器边界。

每个示例都包裹在带背景色的 `View` 中（`show_bg: true, color: theme.color_inset_1`），方便观察图片填充行为。使用相同的 `ducky.png` 图片文件。
