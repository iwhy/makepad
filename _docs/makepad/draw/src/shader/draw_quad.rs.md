# `draw_quad.rs` — 矩形着色器（DrawQuad / DrawColor）

本文件定义了 Makepad 中最核心的 2D 矩形绘制着色器，用于渲染 UI 元素如按钮背景、面板、边框、圆角矩形等。包括 `DrawQuad`（基础矩形）和 `DrawColor`（纯色矩形简化版）两个绘制单元。

## `mod.draw.DrawColor` — 纯色矩形着色器

### 注册

```rust
mod.draw.DrawColor = mod.std.set_type_default() do #(DrawColor::script_shader(vm)){...}
```

最简的矩形渲染，只支持纯色填充。不包含圆角、边框、渐变等复杂功能。用于性能敏感的场景和简单的颜色平铺。

### Vertex 输出 / Fragment 输入

- **`varying_color: vec4`**：从顶点着色器传递的插值颜色。
- **`varying_border_radius: vec4`**：从顶点着色器传递的圆角半径。

### Fragment Shader

1. 接收顶点着色器插值后的 `varying_color` 和 `varying_border_radius`
2. 使用 `Sdf2d.box` 创建圆角矩形 SDF
3. 调用 `sdf_pixel` 计算抗锯齿覆盖度
4. 输出最终颜色

## `mod.draw.DrawQuad` — 完整矩形着色器

### 注册

```rust
mod.draw.DrawQuad = mod.std.set_type_default() do #(DrawQuad::script_shader(vm)){...}
```

这是 Makepad UI 渲染中使用最广泛的着色器，支持圆角、粗细可控的边框、多种阴影效果。

### Vertex Output / Fragment Input Uniforms

- **`color: vec4`**：主要填充颜色，支持 alpha 透明度。
- **`border_color: vec4`**：边框颜色，独立于填充色。
- **`border_radius: vec4`**：四角独立圆角半径 `(top_left, top_right, bottom_right, bottom_left)`。
- **`border_width: float`**：边框宽度（像素单位）。
- **`shadow_color: vec4`**：阴影颜色。
- **`shadow_radius: float`**：阴影圆角半径。
- **`shadow_rect: vec4`**：阴影矩形边界。
- **`inv_shadow_radius: float`**：阴影半径的倒数。
- **`shadow_gradient: vec4`**：高斯阴影纹素。
- **`varying_color: vec4`**、**`varying_border_color: vec4`**：顶点插值颜色。
- **`varying_border_radius: vec4`**、**`varying_border_width: float`**：顶点插值几何。
- **`varying_shadow_color: vec4`**、**`varying_shadow_rect: vec4`**、**`varying_inv_shadow_radius: vec4`**、**`varying_shadow_gradient: vec4`**、**`varying_shadow_radius: float`**：顶点插值阴影数据。

### Fragment Shader — `fn pixel()` 实现

#### 输入

- `varying_border_radius`：四角圆角半径
- `varying_border_width`：边框宽度
- `varying_color` / `varying_border_color`：填充色/边框色
- 阴影相关 uniforms
- `aa_viewport`：抗锯齿视口（继承自 Sdf2d）

#### 执行流程

1. **计算有效绘制区域**：`rect = vec4(0.0, 0.0, aa_viewport)`，即当前片段覆盖的像素矩形
2. **阴影渲染**：如果启用阴影，调用 `GaussShadow.shadow_fn` 计算当前像素的阴影贡献值，乘以 `varying_shadow_color` 的 alpha 通道
3. **SDF 圆角矩形计算**：
   - 使用 `Sdf2d.flat_rounded_box(rect, varying_border_radius)` 创建圆角矩形 SDF
   - 调用 `sdf_pixel` 获取填充颜色的覆盖度
4. **边框计算**（如果 `varying_border_width > 0`）：
   - 计算内部矩形：`inner_rect = rect - varying_border_width * 2`
   - 使用相同圆角半径的内部圆角矩形 SDF
   - 填充覆盖度 = 外部矩形 SDF - 内部矩形 SDF
   - 使用 `varying_border_color` 渲染
5. **颜色混合**：
   - 填充颜色与边框颜色在同一像素上混合（如果边框覆盖了该像素）
   - 阴影颜色在底层混合
   - 最终输出 `color * coverage`，利用 alpha 混合实现抗锯齿

#### 关键技术细节

- 圆角矩形 SDF 使用有符号距离场，正值表示在形状外部（需要抗锯齿过渡）
- 抗锯齿使用 `smoothstep(0.0, aa_width, -distance)` 获得平滑边缘
- 边框在内外边缘都进行抗锯齿处理
- 阴影使用预计算的高斯纹素，在像素着色器中进行一维卷积，避免对每个像素进行昂贵的多重采样

### 图形管线

1. **顶点着色器**：将矩形的四个角传递给片段着色器，计算插值颜色和几何值
2. **片段着色器**：`fn pixel()` 是主要的渲染入口
3. 每个 `DrawQuad` 实例渲染一个矩形，大量矩形可以通过 instancing 批量提交
