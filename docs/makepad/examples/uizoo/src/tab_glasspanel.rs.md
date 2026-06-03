# tab_glasspanel.rs

GlassPanel 玻璃态面板展示页面 — 演示 Frosted Glass 效果的多种参数配置。

## 默认效果

```rust
GlassPanel{
    width: 260  height: 140
    padding: theme.mspace_3
    flow: Down  spacing: theme.space_1
    Label{text: "GlassPanel"}
    Label{text: "Default material"}
}
```

默认玻璃材质面板，包含标题和描述文本。

## 自定义色调 + 边框

```rust
GlassPanel{
    draw_bg +: {
        tint_color: #6af
        tint_alpha: 0.23
        border_color: #9cf
        border_alpha: 0.6
        border_width: 1.5
        corner_radius: 18.0
        specular_strength: 0.5
        noise_strength: 0.025
    }
}
```

自定义参数：
- `tint_color` / `tint_alpha`: 色调颜色和透明度。
- `border_color` / `border_alpha` / `border_width`: 边框颜色、透明度、宽度。
- `corner_radius`: 圆角半径 18px。
- `specular_strength`: 高光强度 0.5。
- `noise_strength`: 噪点强度 0.025。

## 场景模糊

```rust
GlassPanel{
    draw_bg +: {
        tint_color: #fff
        tint_alpha: 0.18
        use_scene_blur: 1.0
        blur_amount: 0.75
        specular_strength: 0.4
        noise_strength: 0.04
    }
}
```

- `use_scene_blur: 1.0`: 启用场景背景模糊（M2 芯片优化）。
- `blur_amount: 0.75`: 模糊程度。
- 模拟 macOS 风格的毛玻璃效果。
