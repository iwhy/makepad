# `draw_svg.rs` — SVG 着色器（DrawSvg）

本文件实现了 SVG 渲染着色器，通过扩展 `DrawVector` 的矢量路径引擎增加颜色覆盖映射功能。支持 SVG 路径的 fill/stroke 颜色替换和渐变覆盖。

## `mod.draw.DrawSvg` — SVG 着色器

### 注册

```rust
mod.draw.DrawSvg = mod.std.set_type_default() do #(DrawSvg::script_shader(vm)){...}
```

### 继承关系

```
DrawVector（基础矢量路径渲染）
  └─ DrawSvg（+ SVG 颜色覆盖）
```

使用 `..mod.draw.DrawVector` 继承所有基础 uniform（gradient_texture、fill_color、stroke_color 等）、vertex 输出和 varying 定义。

### 新增功能

- SVG 颜色覆盖映射（替换或修改 SVG 中的特定颜色）
- 支持多个颜色覆盖层
- 保留原始 SVG 路径的渐变/纹理信息
- 支持 SVG 的继承样式解析

### Uniforms

- **`svg_color_overrides: array<vec4, N>`**：颜色覆盖数组，每项格式为 `(original_color_rgba, replacement_color_rgba)`
- **`svg_color_override_count: float`**：启用的覆盖数量
- **`svg_opacity: float`**：整体不透明度（全局 alpha 覆盖）

### Fragment Shader — `fn pixel()` 执行流程

1. **继承 DrawVector 的像素着色器**：先执行完整的矢量路径渲染（SDF 距离计算、渐变/纹理采样、描边计算）
2. **颜色覆盖映射**：
   - 对每个片段，检查当前计算出的颜色
   - 遍历 `svg_color_overrides` 数组
   - 如果当前颜色匹配某个 `original_color`（在一定容差内），替换为 `replacement_color`
   - 支持模糊匹配（考虑 alpha 混合的影响）
3. **全局透明度**：乘以 `svg_opacity`
4. **输出**：最终颜色 = `base_vector_color.overridden_with_svg_colors * svg_opacity`

### SVG 加载系统

虽然颜色覆盖在 GPU 着色器中完成，SVG 解析在 CPU 端执行：
1. 解析 SVG XML 文件为路径树
2. 提取每个路径元素的 fill/stroke 颜色
3. 构建路径数据传递给 `DrawVector`
4. 构建颜色覆盖映射表上传到 `DrawSvg`

### 颜色覆盖映射表

- 每个覆盖项是一个 8-float 的条目（原始色 RGBA + 替换色 RGBA）
- 最大覆盖数量由 `MAX_SVG_COLOR_OVERRIDES` 常量定义
- 覆盖匹配在 GPU 上进行，按像素评估
- 支持精确匹配（相等检查）和近似匹配（颜色距离检查）

### 使用场景

- 图标主题化（替换 SVG 图标中的特定颜色以匹配主题）
- 交互状态反馈（hover/active 时颜色变化）
- SVG 动画（通过 CPU 更新覆盖映射实现颜色过渡）
- 动态品牌颜色应用

### 与 DrawVector 的关系

- `DrawSvg` 是 `DrawVector` 的功能超集
- 所有 `DrawVector` 的功能（渐变、纹理、描边、dash）在 `DrawSvg` 中保持可用
- `DrawSvg` 增加了颜色覆盖的一层间接性
- 如果不需要颜色覆盖，应使用 `DrawVector` 以获得更好性能
