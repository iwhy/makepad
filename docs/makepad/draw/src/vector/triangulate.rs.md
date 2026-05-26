# `draw/src/vector/triangulate.rs` — 矢量路径三角化与顶点打包

## 文件作用

此文件实现了将矢量路径（`VectorPath`）通过曲面细分引擎转换为 GPU 顶点缓冲区的完整数据管线。它封装了 `makepad_svg::tessellate::Tessellator` 的调用，并提供了将三角化结果组装为 GPU 友好格式的工具函数。

## 常量

### `VECTOR_FLOATS_PER_VERTEX = 19`

每个矢量顶点在累加缓冲区中占用 19 个 `f32` 值（76 字节）。这 19 个值的布局如下：

| 偏移 | 字段 | 用途 |
|------|------|------|
| 0 | `x` | 顶点 X 坐标（世界空间） |
| 1 | `y` | 顶点 Y 坐标 |
| 2 | `u` | 纹理 U 坐标 |
| 3 | `v` | 纹理 V 坐标 |
| 4-7 | `color[r,g,b,a]` | 预乘 alpha 后的 RGBA 颜色 |
| 8 | `stroke_mult` | 描边乘数（用于符号距离场判定） |
| 9 | `stroke_dist` | 描边符号距离值 |
| 10 | `shape_id` | 形状标识符 |
| 11-16 | `param[0..5]` | 6 个通用参数（可重用于阴影参数、渐变参数等） |
| 17 | `clip_radius` | 裁剪半径（GPU 端抗锯齿用） |
| 18 | `zbias` | 深度偏置（画家算法排序用） |

### `VECTOR_ZBIAS_STEP = 0.000001`

相邻形状之间的深度偏置步长。由于矢量渲染使用画家算法（后绘制的形状覆盖先绘制的），每个后续形状的深度值应该略大于前一个，以确保正确的覆盖顺序。

## 数据结构

### `VectorRenderParams` — 渲染参数打包

```rust
pub struct VectorRenderParams {
    pub color: [f32; 4],      // RGBA 颜色
    pub stroke_mult: f32,     // 描边乘数
    pub shape_id: f32,        // 形状 ID
    pub params: [f32; 6],     // 6 个通用参数
    pub zbias: f32,           // 深度偏置
}
```

此结构体封装了写入每个顶点所需的渲染参数。这些参数对所有属于同一形状的顶点是共享的，但在展开过程中被复制到每个顶点记录中，以简化 GPU 端的读取逻辑。

## 核心函数

### `tessellate_path_fill` — 填充路径三角化

**参数**：可变路径引用、可变三角化器引用、顶点缓冲区、索引缓冲区、线段连接类型、miter 限制值、抗锯齿参数、是否使用 GPU 扩展填充。

**实现逻辑**：

1. **路径压平（Flatten）**：调用 `tess.flatten(path, 0.25)`，将路径中的所有曲线段（贝塞尔曲线）递归细分直到误差小于 0.25 像素，生成仅包含直线段的近似多边形。

2. **填充三角化**：调用 `tess.fill(aa, line_join, miter_limit, gpu_expand_fill, tess_verts, tess_indices)`，执行多边形三角化：
   - 默认（`gpu_expand_fill = false`）：使用"fringe"方法，在路径边界添加一个扩展边带（由 `aa` 参数控制宽度），这个边带在 GPU 像素着色器中通过符号距离场计算抗锯齿 alpha。
   - GPU 扩展模式（`gpu_expand_fill = true`）：不预生成抗锯齿边带，而是依靠 GPU 在顶点着色器中扩展边缘，适用于避免某些 GPU（如 Apple Metal）上的共点三角形栅格化问题。

3. **裁剪半径计算**：调用 `compute_clip_radii(tess_verts, tess_indices)`，遍历每个三角形的三个顶点，对每个顶点记录它到同三角形中其他顶点的最大距离。这个值供 GPU 端裁剪/抗锯齿使用。

4. **路径清空**：`path.clear()`，释放已压平的路径数据，允许 DrawVector 重用路径缓冲区。

### `tessellate_path_stroke` — 描边路径三角化

**参数**：可变路径引用、可变三角化器引用、顶点缓冲区、索引缓冲区、描边宽度、线帽类型、线段连接类型、miter 限制值、抗锯齿参数。

**实现逻辑**：

1. 调用 `tess.flatten(path, 0.25)` 压平路径（同填充）。

2. **描边三角化**：调用 `tess.stroke(stroke_width, line_cap, line_join, miter_limit, aa, tess_verts, tess_indices)`，将描边轮廓展开为一组三角形：
   - 对 `line_to` 段：扩展为矩形条带（两侧各偏移 stroke_width/2）。
   - 对 `MoveTo` 段：应用指定的线帽风格（butt：矩形端点；round：半圆端点；square：延长半个线宽的矩形端点）。
   - 对拐角：应用指定的线段连接风格（miter：尖角；round：圆角；bevel：切角）。

3. 计算裁剪半径，清空路径。

4. **返回值**：返回抗锯齿缩放因子 `(stroke_width * 0.5 + aa * 0.5) / aa`，即半宽与抗锯齿边带之和除以抗锯齿宽度。这个值用于 GPU 端将符号距离值转换为实际 alpha。若 `aa == 0`（无抗锯齿），返回一个极大值 `1e6`。

### `append_tessellated_geometry` — 追加三角化几何到累加器

**参数**：顶点切片（`VVertex` 数组）、索引切片（`u32` 数组）、累加顶点缓冲区 `&mut Vec<f32>`、累加索引缓冲区 `&mut Vec<u32>`、渲染参数 `VectorRenderParams`。

**实现逻辑**：

1. **空检查**：若顶点或索引为空，直接返回。

2. **计算基础索引偏移**：`base = acc_verts.len() / VECTOR_FLOATS_PER_VERTEX`，作为当前批次的起始顶点索引（用于调整索引值）。

3. **顶点展开**：遍历每个 `VVertex`，将结构体字段和渲染参数字段按固定布局写入 `acc_verts`：
   ```text
   x, y, u, v,                    // 来自 VVertex
   color[0..=3],                   // 来自 VectorRenderParams
   stroke_mult, stroke_dist,       // stroke_mult 来自参数，stroke_dist 来自 VVertex
   shape_id,                       // 来自 VectorRenderParams
   param[0..=5],                   // 来自 VectorRenderParams
   clip_radius,                    // 来自 VVertex
   zbias                           // 来自 VectorRenderParams
   ```
   每个顶点固定写入 19 个 `f32`。

4. **索引导入**：遍历原始索引，每个索引加上 `base` 偏移后推入 `acc_indices`。这样可以正确地将多个独立三角化批次的索引拼接到统一的索引缓冲区中。

## 设计要点

1. **结构化数组（SoA 还是 AoS）**：此实现使用数组结构体（Array of Structs）布局——每个顶点的所有属性连续存储。这种布局简化了顶点数据的上传逻辑，GPU 着色器可以通过固定偏移访问每个属性。

2. **参数复制 vs 索引化**：颜色、shape_id、zbias 等共享参数被复制到每个顶点中，而非使用独立参数缓冲区加顶点索引的方式。这是 GPU 矢量渲染常见的权衡：复制参数简化了着色器实现（无需额外的顶点属性或 uniform 数组查找），但增加了内存开销。

3. **增量累加模式**：函数设计为可重复调用，每次调用将新的三角化批次追加到累加器末尾。调用方可以一次性构建整个帧的所有填充 + 描边几何，然后统一上传到 GPU。

4. **Z-Occlusion 禁用**：矢量渲染不需要深度测试，zbias 只用于形状间的顺序排列。`VECTOR_ZBIAS_STEP` 确保即使形状数量很多，深度值也不会溢出 32 位浮点的有效精度范围。
