# glass_panel.rs — 玻璃面板（毛玻璃效果）

## 概述
`GlassPanel` 是一个实现毛玻璃/亚克力效果的半透明面板组件。通过高斯模糊背景和半透明覆盖层模拟 macOS 风格或 Windows 亚克力风格的视觉毛玻璃效果。

## 核心结构

### GlassPanel
- **`draw_bg: DrawGlass`**：毛玻璃背景绘制器。
- **`draw_border: DrawQuad`**：可选边框绘制器。
- **`border_visible: bool`**：是否绘制边框。

### DrawGlass
继承自 `DrawGauss` 的高斯模糊绘制器，包含实例级模糊半径和覆盖层颜色。

### DrawGauss
基础高斯模糊绘制器，继承自 `DrawQuad`：
- **`blur_radius: f32`**：高斯模糊半径（像素）。
- **`color: Vec4`**：覆盖着色。
- **`strength: f32`**：覆盖强度（0.0-1.0）。

Shader 实现：在片段 shader 中，先对背景采样进行高斯模糊，再与覆盖层颜色进行混合。使用 `Sdf2d` 做圆角矩形裁剪。

## 核心方法

### Widget 实现

**`draw_walk`**：委托给 `draw_glass_panel`，调用 DrawGlass 绘制毛玻璃效果。

**`handle_event`**：无特殊交互逻辑，仅处理子组件事件。

### 绘制器调用

DrawGlass 的 `draw_abs`/`draw_walk` 方法使用 `cx2d` 中的高斯模糊管线（`gauss_renderer`）在 GPU 上执行高效的框式模糊或两遍高斯模糊。

### GlassPanelRef

`GlassPanelRef` 通过 `borrow`/`borrow_mut` 提供安全的子组件访问。
