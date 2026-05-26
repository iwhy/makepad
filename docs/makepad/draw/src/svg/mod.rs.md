# `draw/src/svg/mod.rs` — SVG 模块入口

## 文件作用

此文件是 `draw` crate 中 SVG 子模块的入口文件。它将外部 SVG 解析库（`makepad-svg`）的核心类型和函数重新导出到 `draw` crate 的公共 API 中，同时声明了内部的 `render` 子模块。

## 结构概览

### `render` 子模块

```rust
pub mod render;
```

声明 `render` 为公共子模块。该模块位于 `svg/render.rs` 中，负责将解析后的 SVG 文档树（`SvgDocument`）遍历并逐节点转换为 `DrawVector` 绘图指令。`render_svg` 函数是该模块的唯一公共入口。

### 从 `makepad-svg` 重新导出的类型和函数

| 导出项 | 来源 | 用途 |
|-------|------|------|
| `makepad_svg::animate` | `makepad-svg` crate | SVG 动画求值模块，包含颜色、浮点、路径路径和变换四种动画类型的运行时求值 |
| `makepad_svg::document::*` | `makepad-svg` crate | SVG 文档模型的所有公共类型：`SvgDocument`、`SvgNode`、`SvgGroup`、`SvgPath`、`SvgRect`、`SvgCircle`、`SvgEllipse`、`SvgLine`、`SvgPolyline`、`SvgPolygon`、`SvgUse`、`SvgDefs`、`SvgStyle`、`SvgGradient`、`SvgFilter` 等 |
| `parse_svg` | `makepad-svg` crate | 将 SVG XML 文本解析为 `SvgDocument` 树结构 |
| `LineCap`, `LineJoin` | `makepad-svg` crate | 描边样式枚举：线帽（butt/round/square）和线段连接（miter/round/bevel） |
| `parse_path_data` | `makepad-svg` crate | 解析 SVG 路径的 `d` 属性字符串（如 `M10 10 L20 20`）为 `VectorPath` 指令序列 |
| `parse_transform` | `makepad-svg` crate | 解析 SVG 变换字符串（如 `translate(10,20) scale(2)`）为 `Transform2d` |
| `viewbox_transform` | `makepad-svg` crate | 根据 viewBox 和视口尺寸计算缩放平移参数 `(sx, sy, tx, ty)` |
| `render_svg` | 本 crate 的 `render` 子模块 | **核心渲染入口**，驱动整个 SVG 到 `DrawVector` 的转换过程 |

## 设计说明

1. **分层架构**：`svg/mod.rs` 作为薄层入口，将解析（`parse_svg`）和渲染（`render_svg`）清晰分离。解析层在 `makepad-svg` 外部 crate 中，渲染层在 `svg/render.rs` 中。

2. **类型重用**：所有 SVG 文档模型类型（`SvgNode`、`SvgGroup` 等）均直接来自 `makepad-svg`，避免在 `draw` crate 中维护冗余的 SVG 类型定义。

3. **动画支持**：`makepad_svg::animate` 模块的重新导出使上层调用方可以访问动画求值功能，尽管实际的动画求值发生在 `render.rs` 内部。

4. **无额外逻辑**：该文件仅做重新导出和模块声明，不包含任何业务逻辑或数据结构定义。
