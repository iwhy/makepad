# `mod.rs` — Draw 着色器模块索引

本文件是 Makepad `draw` crate 着色器子系统的模块注册入口，通过 `pub mod` 声明将 11 个独立的着色器模块暴露给外部调用。

## 模块列表

| 模块 | 对应文件 | 用途 |
|------|----------|------|
| `sdf` | `sdf.rs` | SDF 绘图原语：高斯阴影、调色板函数、`Sdf2d` 结构体 |
| `draw_quad` | `draw_quad.rs` | 矩形着色器：背景、边框、圆角、SDF 渲染 |
| `draw_text` | `draw_text.rs` | 文本着色器：字体样式、slug 系统、UV / glyph 渲染 |
| `draw_glyph` | `draw_glyph.rs` | Glyph 渲染着色器：曲线纹理、带状纹理 |
| `draw_cube` | `draw_cube.rs` | 3D 立方体着色器：顶点光照、透视投影 |
| `draw_pbr` | `draw_pbr.rs` | PBR 着色器：glTF 材质、环境光照、IBL、阴影 |
| `draw_vector` | `draw_vector.rs` | 矢量路径着色器：渐变纹理、stroke/fill |
| `draw_svg` | `draw_svg.rs` | SVG 着色器：扩展 DrawVector，颜色覆盖 |
| `draw_svg_glyph` | `draw_svg_glyph.rs` | SVG 字形着色器：扩展 DrawGlyph |
| `draw_rotated_text` | `draw_rotated_text.rs` | 旋转文本着色器：扩展 DrawText，旋转矩阵 |
| `draw_text_3d` | `draw_text_3d.rs` | 3D 文本着色器：扩展 DrawRotatedText，广告牌/面向相机模式 |

## 架构关系

- **基础着色器**：`draw_quad`（矩形）、`draw_cube`（立方体）、`draw_glyph`（字形）是三大基础 GPU 绘制原语
- **文本着色器链**：`draw_text` → `draw_rotated_text`（+旋转） → `draw_text_3d`（+广告牌）
- **矢量着色器链**：`draw_vector`（路径引擎） → `draw_svg`（颜色覆盖）
- **Glyph 着色器链**：`draw_glyph` → `draw_svg_glyph`（SVG 字形）
- **共享原语**：所有着色器都依赖 `sdf` 模块中的 `Sdf2d` 结构体和 `GaussShadow` 函数

## 注册方式

每个模块在 `script_mod!` 宏中通过以下模式注册到 `mod.draw.*` 命名空间：

```rust
mod.draw.DrawXxx = mod.std.set_type_default() do #(DrawXxx::script_shader(vm)){...}
```

对于扩展着色器，使用 `..mod.draw.BaseXxx` 语法继承基础着色器的 uniform/varying/vertex 定义。
