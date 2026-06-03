# tab_view.rs

View 容器展示页面 — 演示各种 View 变体的用途。

## 基础 View

```rust
View{
    width: Fit height: Fit
    padding: theme.mspace_2  align: Align{x: 0.5 y: 0.5}
    Label{text: "View"}
}
```

## View 样式变体

### SolidView

```rust
SolidView{
    draw_bg +: {color: #F00}
    Label{text: "SolidView"}
}
```

纯色背景的视图容器。

### RoundedView

```rust
RoundedView{
    draw_bg +: { color: #F00  border_radius: 5.0  border_size: 2.0  border_color: #FFF }
    Label{text: "RoundedView"}
}
```

圆角背景视图，支持边框。

### CircleView

```rust
CircleView{ Label{text: "CircleView\nFit"} }
CircleView{ padding: 30  Label{text: "CircleView\nFit Pad 30"} }
CircleView{ width: 60 height: 60  padding: 0 }
```

圆形视图，展示了不同尺寸和内边距下的形态。

### ScrollXYView

```rust
ScrollXYView{
    width: 100 height: 100
    View{ width: 400. height: 400. ... }  // 内部大视图
}
```

双向滚动容器，内含 400x400 的内容在 100x100 的视口中可滚动。

### ScrollYView

```rust
ScrollYView{
    width: 100 height: 100
    View{ width: 400. height: 400. ... }  // 水平溢出被裁剪
}
```

垂直单向滚动容器，水平方向溢出被裁剪。

## 特殊功能

### CachedView

```rust
CachedView{
    View{ show_bg: true  draw_bg +: {color: uniform(theme.color_inset)}
          Label{text: "CachedView"} }
}
```

缓存渲染结果的视图容器，子视图不变时不重绘，适合静态内容优化性能。
