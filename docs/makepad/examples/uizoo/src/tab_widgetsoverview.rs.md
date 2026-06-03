# tab_widgetsoverview.rs

UI Zoo 欢迎/概述页面 — Makepad 框架介绍。

```rust
mod.widgets.WidgetsOverview = View{
    spacing: theme.space_2  padding: theme.mspace_2
    flow: Down  align: Align{x: 0.5 y: 0.5}
    height: Fill width: Fill

    ScrollYView{
        flow: Down  width: 430. height: Fill
        align: Align{x: 0.0 y: 0.4}  spacing: theme.space_3
```

- `ScrollYView`: 内容可垂直滚动，宽度固定 430px，居中显示。

### 内容

```rust
Image{src: crate_resource("self:resources/logo_makepad.png") fit: ImageFit.Biggest}
```

Makepad Logo 图片，`Biggest` 模式等比缩放。

```rust
H4{text: "Makepad is an open-source, cross-platform UI framework written in and for Rust."}
P{text: "Built on a shader-based architecture, Makepad delivers high performance..."}
P{text: "One of Makepad's standout features is live styling..."}
P{text: "This example application provides an overview of the currently supported widgets."}
TextBox{text: "UI Zoo hosts a high number of widgets and variants..."}
```

概述文本：
- Makepad 是基于 Rust 的开源跨平台 UI 框架。
- 支持 Windows/Linux/macOS/iOS/Android/Web。
- 基于着色器架构，适合复杂应用甚至 3D/VR/AR。
- 核心特性：Live Styling（热更新 UI 样式无需重编译）。
- UI Zoo 展示了所有支持的 Widget。
- 免责声明：加载时间因 Widget 数量较多而不能代表典型应用。
