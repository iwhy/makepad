# tab_adaptiveview.rs

AdaptiveView 自适应布局展示页面 — 演示根据屏幕宽度切换布局变体。

## 子视图定义

```rust
let ViewA = RoundedView{
    width: 200 height: Fill  show_bg: true  draw_bg.color: #176951
    Label{text: "View A"}
}

let ViewB = RoundedView{
    width: Fill height: Fill  show_bg: true  draw_bg.color: #1f3a67
    flow: Down  spacing: 5.  Label{text: "View B"}
}
```

两个子视图：ViewA 固定宽度 200px（绿色），ViewB 填充剩余宽度（蓝色）。

## AdaptiveView 配置

```rust
AdaptiveView{
    Desktop := View{
        flow: Right  align: Align{x: 0.0 y: 0.5}  spacing: 20.
        ViewA{}  ViewB{}
    }

    Mobile := View{
        flow: Down  align: Align{x: 0.5 y: 0.0}  spacing: 10.
        ViewA{}  ViewB{}
    }
}
```

- `Desktop` 变体：水平并排（ViewA 在左，ViewB 在右）。
- `Mobile` 变体：垂直堆叠（ViewA 在上，ViewB 在下）。
- 默认断点：宽度 < 860px 切换为 Mobile 布局。

容器使用 `RoundedView` 包裹并设置背景色，方便观察布局切换效果。说明文字提示用户调整窗口宽度观察变化。
