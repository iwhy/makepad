# tab_align_scroll.rs

对齐 + 滚动展示页面 — 测试对齐和滚动同时工作时的行为。

## 背景

当子视图超出容器大小时，居中对齐可能导致负偏移与滚动行为冲突。此 Demo 验证修复：溢出时对齐偏移被 clamp 到零，使滚动正常。

## 辅助模板

```rust
let AlignScrollBox = RoundedView{ show_bg: true draw_bg.color: #x0F02 border_size: 1. }
let ScrollContainer = SolidView{ draw_bg.color: (#x1a1a2e) }
```

## Flow: Down + 垂直居中 + 垂直滚动

```rust
ScrollContainer{
    width: Fill height: 200.
    flow: Down  align: Align{x: 0.5 y: 0.5}
    scroll_bars: ScrollBars{ show_scroll_y: true }
    AlignScrollBox{width: 120. height: 80. P{text: "Box 1"}}
    // ... Box 2~5
}
```

内容有限时居中；超出容器高度时可向下滚动。

## Flow: Down + 水平居中 + 水平滚动

```rust
ScrollContainer{
    width: 250. height: Fit
    flow: Down  align: Align{x: 0.5 y: 0.0}
    scroll_bars: ScrollBars{ show_scroll_x: true }
    AlignScrollBox{width: 400. height: 40. P{text: "Wide box 1 — 400px in a 250px container"}}
    // 宽项超出时水平滚动
}
```

## Flow: Right + 垂直居中 + 垂直滚动

```rust
ScrollContainer{
    height: 150.  flow: Right  align: Align{x: 0.0 y: 0.5}
    scroll_bars: ScrollBars{ show_scroll_y: true }
    AlignScrollBox{height: 250. P{text: "Tall (250px)"}}  // 超出垂直空间
}
```

## Flow: Right + 水平居中 + 水平滚动

```rust
ScrollContainer{
    width: 300.  flow: Right  align: Align{x: 0.5 y: 0.5}
    scroll_bars: ScrollBars{ show_scroll_x: true }
    AlignScrollBox{width: 100. P{text: "A"}} // ... B~E 超出水平空间
}
```

## Flow: Overlay + 居中 + 双向滚动

```rust
ScrollContainer{
    width: 250. height: 150.  flow: Overlay
    align: Align{x: 0.5 y: 0.5}
    scroll_bars: ScrollBars{ show_scroll_x: true show_scroll_y: true }
    AlignScrollBox{width: 400. height: 300. P{text: "400x300 box in a 250x150 container"}}
}
```

覆盖布局中超大子项的双向滚动。

## 内容适应时居中

```rust
ScrollContainer{
    width: Fill height: 150.  flow: Down  align: Align{x: 0.5 y: 0.5}
    AlignScrollBox{width: 120. height: 40. P{text: "Centered"}}
}
```

验证当内容小于容器时，居中对齐正常工作。
