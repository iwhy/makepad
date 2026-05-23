# `draw_glyph.rs` — Glyph 渲染着色器（DrawGlyph）

本文件定义了 Makepad 的 GPU glyph 渲染着色器，使用基于曲线纹理和带状纹理的 SDF 技术，支持高质量字形渲染。纹理压缩和 GPU 加速的 SDF 计算是其核心特色。

## `mod.draw.DrawGlyph` — Glyph 着色器

### 注册

```rust
mod.draw.DrawGlyph = mod.std.set_type_default() do #(DrawGlyph::script_shader(vm)){...}
```

### 架构概述

`DrawGlyph` 采用两级纹理系统和三段式 SDF 重建算法：
1. **Curve Texture（曲线纹理）**：存储字形轮廓的曲线参数，用于计算像素到曲线段的最近距离
2. **Band Texture（带状纹理）**：存储三频带 SDF 数据，粗-中-细三个分辨率
3. **最终像素着色器**：结合曲线距离和带状距离输出高质量抗锯齿字形

### Vertex Output / Fragment Input

- **`varying_color: vec4`**：顶点色（插值后的文字颜色）
- **`varying_glyph_pos: vec2`**：字形在屏幕空间的位置
- **`varying_glyph_scale: vec2`**：字形缩放
- **`varying_uv_bias: vec4`**：UV 偏移和缩放
- **`varying_glyph_texture_index: float`**：纹理数组索引
- **`varying_has_color_glyph: float`**：彩色 glyph 标志
- **`varying_has_thin_lines: float`**：细线优化标志

### Fragment Shader — `fn pixel()` 执行流程

1. **计算局部坐标**：使用 `varying_glyph_pos`、`varying_glyph_scale` 和片段位置计算局部 UV 坐标
2. **曲线纹理采样**：
   - 从曲线纹理中采样当前 UV 附近的曲线段
   - 解码曲线参数（直线/二次/三次贝塞尔曲线控制点）
   - 计算像素到所有曲线段的有符号距离
   - 选择最短有符号距离作为曲线 SDF 值

3. **带状纹理采样**：
   - 从三频带纹理的 R/G/B 通道分别采样三个分辨率级别的 SDF 值
   - 低频带（R 通道）：粗粒度距离场，覆盖整个字形范围
   - 中频带（G 通道）：中等分辨率，覆盖 glyph 边缘附近
   - 高频带（B 通道）：高分辨率细节，仅在边缘附近生效

4. **SDF 重建**：
   - 将曲线 SDF 与三频带 SDF 进行加权融合
   - 远距离使用带状 SDF（曲线 SDF 在大距离时不精确）
   - 近距离使用曲线 SDF（提供亚像素精度）
   - 融合权重根据距离值渐变过渡

5. **抗锯齿输出**：
   - 对重建的 SDF 值应用 `smoothstep`
   - 边缘宽度 = 1 像素（根据 `aa_viewport` 计算）
   - 输出颜色 = `varying_color * coverage`

6. **特殊处理**：
   - **彩色字形**：如果 `varying_has_color_glyph > 0`，不使用 `varying_color` 而是直接使用纹理中采样的颜色数据（用于 emoji/彩色字体）
   - **细线优化**：如果 `varying_has_thin_lines > 0`，调整抗锯齿宽度以保留细线细节，避免细线被抗锯齿过度平滑消失

### Curve Texture 详细说明

曲线纹理以纹素形式编码字形轮廓：
- 每个纹素存储 4 个 float（RGBA），编码曲线段的类型和参数
- **直线段**：编码起点和终点坐标
- **二次贝塞尔**：编码起点、控制点和终点
- **三次贝塞尔**：编码起点、两个控制点和终点（占用两个纹素）
- 曲线数据在栅格化时从纹理读取，避免在 GPU 上存储大量顶点缓冲

### Band Texture 详细说明

三频带 SDF 是 Makepad 的专有技术：
- **低频带（R）**：低分辨率 SDF，覆盖 glyph 边界外的大范围距离（下降缓慢）
- **中频带（G）**：中等分辨率 SDF，覆盖 glyph 边界附近的过渡区
- **高频带（B）**：高分辨率 SDF，仅在 glyph 边界极窄范围内有效
- 三者组合可在低纹理带宽下实现高质量的缩放渲染
- 相比传统单频带 SDF，三频带减少了放大时的边缘阶梯和缩小时的细节丢失

### 与其他着色器的关系

- `DrawGlyph` 是基础 glyph 着色器
- `draw_svg_glyph.rs` 中的 `DrawSvgGlyph` 通过 `..mod.draw.DrawGlyph` 继承并扩展 SVG 颜色覆盖功能
