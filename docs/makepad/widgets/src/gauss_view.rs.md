# gauss_view.rs — 高斯模糊背景视图

## 概述
`GaussView` 是一个带高斯模糊背景的视图组件，与 `GlassPanel` 功能类似但更轻量。适用于需要模糊背景但不需要毛玻璃覆盖层的场景，如弹出菜单或浮动面板的背景模糊。

## 核心结构

### GaussView
- **`draw_bg: DrawGauss`**：高斯模糊绘制器，通过 `#[redraw]` 每次重绘。
- **`draw_border: DrawQuad`**：可选边框绘制器。

### DrawGauss
同 `glass_panel.rs` 中的高斯模糊绘制器，包含 `blur_radius`、`color` 和 `strength` 等实例字段。

## 核心方法

### Widget 实现

**`draw_walk`**：调用 `draw_gauss.draw_walk` 绘制模糊背景，然后绘制可选的边框（`draw_border.draw_walk`）。不需要绘制子组件内容，因此不调用 `begin_turtle`/`end_turtle`。

**`handle_event`**：无特殊交互逻辑。

### GaussViewRef

`GaussViewRef` 通过 `borrow`/`borrow_mut` 提供安全的组件访问。

## 与 GlassPanel 的区别

- **GlassPanel**：半透明覆盖层 + 高斯模糊 = 毛玻璃效果。适合按钮、通知等毛玻璃卡片。
- **GaussView**：纯高斯模糊，不添加额外覆盖层。适合菜单背景或临时浮动 UI 的背景模糊。
