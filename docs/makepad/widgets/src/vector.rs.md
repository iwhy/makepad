# vector.rs — 矢量图形绘制引擎

## 概述
`Vector` 组件使用 Makepad 自身的 SDF 矢量绘制管线渲染路径/形状。与 `Svg` 不同，`Vector` 直接操作低级矢量绘制命令（路径、变换、填充、描边），是 DrawVector 和 DrawSvg 的父级基础设施。本文件包含矢量绘制的 Rust 端和 shader 端核心实现。

## 核心结构

### DrawVector
- **`#[deref] draw_super: DrawQuad`**：继承自 DrawQuad。
- **`#[live] svg_data: Option<ScriptHandleRef>`**：SVG 数据句柄。
- **`#[live] svg_size: Vec2`**：SVG 原始尺寸。
- **`#[live] mode: DrawVectorMode`**：缩放模式（`Fit`/`Fill`）。
- **`#[live] dpi: f32`**：DPI 缩放。
- **`#[rust] svg: Option<ScriptHandleRef>`**：运行时 SVG 句柄。

### DrawVectorMode
枚举：`Fit`（保持宽高比内嵌）、`Fill`（保持宽高比充满）。

### DrawVectorTexture
辅助类型，包含 `Vec2`（纹理区域）、`fn pixel(self) -> vec4`（纹理采样 shader）、`fn uniform_block(self) -> vec4`（统一块向量）。

## Shader 实现

### `fn vertex`（顶点 shader）
标准四边形顶点处理，传递位置、矩形、缩放和均匀块到片段 shader。

### `fn pixel`（片段 shader）
核心绘制入口：调用 `Cx2d.path_pixel(self)` 执行 SDF 路径渲染。

## 核心方法

### DrawVector 绘制

**`draw_abs`**：在绝对坐标下绘制矢量。获取 VectorCache 条目，计算模式和缩放。`Fit` 模式下居中显示，`Fill` 模式下完全填充（可能裁剪）。调用 `cx2d.nvg_path_image` 将 NanoVG 路径缓存绑定到绘制上下文。

**`draw_walk`**：返回 `DrawStep::done()` 或委托给 `draw_abs`。当区域尺寸为 0（未布局完成）时跳过绘制。

### DrawSvg

**`draw_abs`**/`draw_walk`**：Svg 的专用绘制方法，在 DrawVector 基础上增加 `tint` 和 `colorize` 覆盖逻辑。

### NVG 路径存储

NanoVG 路径缓存通过 FBO（帧缓冲对象）在 GPU 上实现。`Cx2d.nvg_path_image` 将缓存的路径图像转换为纹理，绑定到当前绘制过程的采样器中。路径缓存的 key 由 SVG 数据句柄的指针构成。

### Vector 组件

`Vector` widget 直接委托给 `DrawVector`，在 Widget trait 的 `draw_walk` 中调用 `self.draw_vector.draw_walk`。

## DrawVector 字段布局注意事项

`DrawVector` 包含 `#[deref] draw_super: DrawQuad`（包含 DrawVars），其后是 `#[live] svg_data`（Option 类型，非 GPU 实例数据）。根据 AGENTS.md 规则 17，非实例字段位于 deref 之后——但这与规则中的"非实例数据应在 deref 之前"相矛盾。实际情况可能是：DrawQuad 的 DrawVars 使用 `as_slice()` 时只读取固定数量的字段，`svg_data: Option<ScriptHandleRef>` 作为 `Option` 类型不会被错误读取。或者这个模式在新版中已被修正。
