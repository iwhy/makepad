# tab_spinner.rs

LoadingSpinner 加载指示器展示页面。

```rust
mod.widgets.DemoSpinner = UIZooTabLayout_B{
    desc +: {}
    demos +: {
        H4{text: "Default"}
        LoadingSpinner{}
    }
}
```

- `desc +: {}`: 描述栏为空（无说明文档）。
- `LoadingSpinner{}`: 使用默认配置的加载旋转动画组件。
- 这是 UI Zoo 中最简洁的 Demo 页面之一，仅展示基本用法。
