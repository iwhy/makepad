# `draw/src/vector/mod.rs` — 矢量图形模块入口

## 文件作用

此文件是 `draw` crate 中矢量图形子模块的入口。它重新导出来自 `makepad-svg` crate 的矢量相关类型和函数，并声明内部的 `triangulate` 子模块，该子模块提供了将矢量路径三角化并将顶点数据打包到 GPU 缓冲区的工具函数。

## 重新导出

### 从 `makepad_svg::paint` 导出

```rust
pub use makepad_svg::paint::*;
```

矢量绘制颜色/样式相关的所有类型，包括：
- `VectorPaint` 枚举：定义填充和描边的颜色/渐变风格，包含 `Solid`（纯色）、`LinearGradient`（线性渐变）、`RadialGradient`（径向渐变）等变体。
- `VectorPaint::solid()` 和 `set_color()` 等构造方法。

### 从 `makepad_svg::path` 导出

```rust
pub use makepad_svg::path::*;
```

矢量路径相关的所有类型，包括：
- `VectorPath`：可变路径数据结构，包含 `Vec<PathCmd>` 命令列表。
- `PathCmd` 枚举：`MoveTo`、`LineTo`、`BezierTo`、`Close`、`Winding`。
- `LineCap` 枚举：`Butt`、`Round`、`Square`。
- `LineJoin` 枚举：`Miter`、`Round`、`Bevel`。

### 从 `makepad_svg::tessellate` 导出

```rust
pub use makepad_svg::tessellate::*;
```

曲面细分（多项式三角化）引擎的所有公共类型，包括：
- `Tessellator`：三角化器，将路径压平（flatten）为线段并执行填充和描边三角化。
- `VVertex`：矢量顶点结构，包含位置 `(x,y)`、纹理坐标 `(u,v)`、符号距离场值 `(stroke_dist)`、裁剪半径 `(clip_radius)` 等字段，由三角形剖分器输出。
- `compute_clip_radii`：计算每个顶点的裁剪半径（clip radius），即顶点到其共享三角形中其他顶点的最大距离，用于 GPU 端抗锯齿。

## 内部子模块

### `triangulate` 子模块

```rust
mod triangulate;
pub use triangulate::*;
```

声明 `triangulate` 为私有模块但重导出其所有公共项。该模块（`vector/triangulate.rs`）提供了三个主要函数：

1. **`tessellate_path_fill`**：执行路径的填充三角化，将路径压平为线段后调用 `Tessellator::fill`，然后计算裁剪半径，清空路径。
2. **`tessellate_path_stroke`**：执行路径的描边三角化，调用 `Tessellator::stroke`，返回抗锯齿缩放因子。
3. **`append_tessellated_geometry`**：将三角化后的顶点和索引数据合并到累加缓冲区中，填充每个顶点的 19 个 `f32` 属性（颜色、描边参数、形状 ID、6 个通用参数、裁剪半径、zbias 等），并调整索引偏移。

### 常量

```rust
pub const VECTOR_FLOATS_PER_VERTEX: usize = 19;
pub const VECTOR_ZBIAS_STEP: f32 = 0.000001;
```

- `VECTOR_FLOATS_PER_VERTEX`：每个矢量顶点在 GPU 缓冲区中占用 19 个 f32（76 字节）。
- `VECTOR_ZBIAS_STEP`：相邻形状之间的深度偏置步长，用于画家算法排序。

## 设计说明

1. **外观模式**：`mod.rs` 作为外观（Facade），将三角化过程所需的全部类型聚合到一个统一的命名空间。调用方只需 `use makepad_widgets::draw::vector::*` 即可访问所有相关类型。

2. **分层分工**：`makepad-svg` 提供底层解析和三角化算法，`triangulate.rs` 提供上层打包和组织逻辑，`DrawVector`（定义在 `shader` 模块中）提供绘画状态的封装。

3. **数据流**：SVG 解析 → 路径构建 → 压平三角化 → 数据打包 → GPU 上传，`mod.rs` 的类型导出使每一层都能访问所需的数据结构。
