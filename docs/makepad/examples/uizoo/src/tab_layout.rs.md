# tab_layout.rs

布局系统演示页面 — 展示 width/height/margin/padding/spacing/flow/align 的基础用法。

## 辅助模板

```rust
let Box = RoundedView{
    show_bg: true
    draw_bg +: { color: uniform(#x0F02) border_size: uniform(1.) border_radius: uniform(0.)
                 border_color: uniform(#xfff8) }
    padding: 3.
    align: Align{x: 0.5 y: 0.5}
}

let BoxLabel = P{ width: Fit align: Align{x: 0.5} }
```

- `Box`: 半透明背景、白色边框的矩形，用于可视化布局。
- `BoxLabel`: 文字居中显示盒子信息。

## Width & Height

```rust
Box{width: 100. height: 60.}          // 固定宽高
Box{width: 100. height: Fill}         // 填充父容器高度
Box{width: 150. height: Fit}           // 自适应内容高度
```

## Margin

```rust
Box{margin: 0.}
Box{margin: 10.}
Box{margin: Inset{top: 0. left: 40 right: 0 bottom: 0.}}
```

展示不同 margin 值的效果。`Inset` 可单独控制四边。

## Padding

```rust
Box{padding: 20.}
Box{padding: Inset{left: 40. right: 10.}}
```

对比 padding 均匀与不均匀设置。

## Spacing

```rust
UIZooRowH{spacing: 10. ...}
UIZooRowH{spacing: 30. ...}
```

子元素间距对比。

## Flow Direction

```rust
UIZooRowH{ flow: Right spacing: 10. ... }  // 水平排列
UIZooRowH{ flow: Down spacing: 10. ... }    // 垂直排列
```

`flow: Right` vs `flow: Down` 的布局差异。

## Align

展示六种对齐组合：

| x | y | 效果 |
|---|---|---|
| 0.0 | 0.0 | 左上 |
| 0.0 | 0.5 | 左中 |
| 0.0 | 1.0 | 左下 |
| 0.5 | 0.0 | 中上 |
| 1.0 | 1.0 | 右下 |

`Align{x: 0.5, y: 0.5}` 为居中。值范围 [0.0, 1.0] 对应起始到结束。
