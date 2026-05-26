# `draw_svg_glyph.rs` — SVG 字形着色器（DrawSvgGlyph）

本文件实现了 SVG 字形的渲染着色器，扩展 `DrawGlyph` 使其支持加载 SVG 格式的字形轮廓（如彩色字体和可缩放字形）。

## `mod.draw.DrawSvgGlyph` — SVG 字形着色器

### 注册

```rust
mod.draw.DrawSvgGlyph = mod.std.set_type_default() do #(DrawSvgGlyph::script_shader(vm)){...}
```

### 继承关系

```
DrawGlyph（基础 glyph 着色器：曲线纹理 + 三频带 SDF）
  └─ DrawSvgGlyph（+ SVG 字形路径渲染）
```

使用 `..mod.draw.DrawGlyph` 继承所有 glyph 渲染的基础设施。当字形包含 SVG 轮廓数据时，`DrawSvgGlyph` 使用矢量路径引擎渲染而非传统的曲线纹理+频带 SDF 方法。

### 核心设计

- 标准 TTF/OTF 字形：回退到 `DrawGlyph` 的曲线纹理+三频带 SDF 渲染
- SVG 字形：使用 `DrawVector`/`DrawSvg` 的路径渲染引擎
- 运行时自动选择渲染路径

### Fragment Shader — `fn pixel()` 执行流程

1. **字形类型判断**：
   - 检查当前 glyph 是否为 SVG 字形（通过 `varying_has_color_glyph` 或专用标志）
   - 非 SVG 字形：直接跳转到 `DrawGlyph` 的标准像素着色器

2. **SVG 字形渲染**：
   - 使用 UV 坐标从 SVG 纹理中采样字形轮廓数据
   - 将 SVG 路径数据转换为 `DrawVector` 兼容的路径段
   - 对路径段执行 SDF 距离计算
   - 支持 SVG 字形的 fill/stroke 颜色（从字体文件中提取）
   - 支持彩色 emoji 的直接颜色输出

3. **彩色字形支持**：
   - SVG 字形通常包含颜色信息（如彩色 emoji）
   - 颜色直接从 SVG 纹理采样，不使用 `varying_color`
   - 支持分层的 SVG 内容（多个填充区域不同颜色）

4. **抗锯齿**：
   - 与 `DrawGlyph` 相同的抗锯齿策略
   - 在 SVG 路径边缘使用 `aa_viewport` 控制过渡宽度

### Uniforms

- **继承自 `DrawGlyph`**：所有 glyph 渲染参数、曲线纹理、频带纹理
- **SVG 特有**：SVG 字形路径纹理、SVG 字形颜色数据

### Vertex Buffer 扩展

在 `DrawGlyph` 的基础上，`DrawSvgGlyph` 可能需要额外的顶点属性来编码 SVG 字形的元数据（如字形中的分层数量、每层的颜色等）。

### 使用场景

- 彩色字体渲染（如 OpenType-SVG 字体）
- Emoji 渲染（彩色 emoji 字体，如 Apple Color Emoji）
- 可缩放图标字体（SVG-in-OpenType 格式）
- 需要保持矢量精度的超大字形渲染

### 与 DrawGlyph 的关系

- `DrawSvgGlyph` 是 `DrawGlyph` 的特殊化扩展
- 两者共享相同的 glyph 管理基础设施（字体纹理图集、glyph 缓存等）
- `DrawSvgGlyph` 仅在字形包含 SVG 数据时激活特殊路径
- 性能特性取决于字形类型：SVG 字形渲染成本通常高于标准 SDF glyph
