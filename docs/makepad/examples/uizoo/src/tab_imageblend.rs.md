# tab_imageblend.rs

ImageBlend 图像混合展示页面 — 在两幅图像间切换。

```rust
mod.widgets.DemoImageBlend = UIZooTabLayout_B{
    desc +: { Markdown{body: "# ImageBlend\n\nImageBlend blends between two images."} }
    demos +: {
        H4{text: "Standard"}
        blendbutton := Button{text: "Blend Image"}
        blendimage := ImageBlend{
            align: Align{x: 0.0 y: 0.0}
            image_a +: { src: crate_resource("self:resources/ducky.png") fit: ImageFit.Smallest }
            image_b +: { src: crate_resource("self:resources/ismael-jean-....jpg") fit: ImageFit.Smallest }
        }
    }
}
```

- `ImageBlend` 包含两幅子图像（`image_a` 和 `image_b`），显示时在两者之间混合过渡。
- `blendbutton`: 在 app.rs 中响应点击：
  ```rust
  if self.ui.button(cx, ids!(blendbutton)).clicked(&actions) {
      self.ui.image_blend(cx, ids!(blendimage)).switch_image(cx);
  }
  ```
  每次点击切换显示 `image_a` 或 `image_b`。
