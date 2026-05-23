# `draw_rotated_text.rs` — 旋转文本着色器（DrawRotatedText）

本文件实现了支持任意角度旋转的文本着色器，通过扩展 `DrawText` 增加旋转矩阵变换。支持 UI 中的倾斜/旋转文本元素。

## `mod.draw.DrawRotatedText` — 旋转文本着色器

### 注册

```rust
mod.draw.DrawRotatedText = mod.std.set_type_default() do #(DrawRotatedText::script_shader(vm)){...}
```

### 继承关系

```
DrawText（基础文本着色器：slug 系统、three-band/UVE/灰度纹理渲染）
  └─ DrawRotatedText（+ 旋转变换）
```

使用 `..mod.draw.DrawText` 继承所有文本渲染基础设施，包括 TextStyle、slug 排版、glyph 渲染管道和 SDF 抗锯齿。

### 新增功能

- 任意角度的文本旋转（0-360 度）
- 旋转中心控制
- 保留文本的原始排版位置

### Uniforms

- **`rotate_angle: float`**：旋转角度（弧度）
- **`rotate_origin: vec2`**：旋转中心（相对于文本包围盒的偏移）
- **`rotate_scale: vec2`**：旋转后的缩放补偿

### 旋转矩阵计算

2D 旋转矩阵（片段着色器中计算）：

```
cos_a = cos(rotate_angle)
sin_a = sin(rotate_angle)
rotated_uv.x = uv.x * cos_a - uv.y * sin_a
rotated_uv.y = uv.x * sin_a + uv.y * cos_a
```

### Fragment Shader — `fn pixel()` 执行流程

1. **UV 计算入口**：
   - 接收从顶点着色器传递的原始 `varying_uv`（未旋转的纹理坐标）
   
2. **旋转应用**：
   - 如果 `rotate_angle != 0.0`：
     - 将 UV 坐标平移到旋转中心：`centered_uv = varying_uv - rotate_origin`
     - 应用 2D 旋转矩阵：`rotated_uv = rotate_matrix * centered_uv`
     - 平移回原始坐标：`final_uv = rotated_uv + rotate_origin`
   - 否则：`final_uv = varying_uv`（不变）

3. **文本渲染**：
   - 使用旋转后的 UV 坐标 `final_uv` 采样 glyph 纹理
   - 所有后续 glyph 渲染逻辑（SDF 计算、抗锯齿、颜色输出）与 `DrawText` 完全相同
   - 相当于在字形空间中对 UV 坐标施加逆向旋转，使渲染结果呈现旋转效果

### 旋转特性

- 旋转不影响文本的排版计算（Slug 在 CPU 端以未旋转的方式排版）
- 旋转发生在 GPU 的纹理采样阶段
- 旋转中心可任意指定（常见为文本中心或左上角）
- 缩放补偿 `rotate_scale` 用于在旋转后保持像素密度

### 使用场景

- UI 中的旋转标签
- 倾斜的按钮文字
- 艺术性文字布局
- 角度标注器

### 与 DrawText 的关系

- `DrawRotatedText` 完全兼容 `DrawText` 的所有功能
- 在不需要旋转时，使用 `DrawText` 可避免旋转矩阵的计算开销
- 旋转功能通过 uniform 控制，不会增加额外的纹理或缓冲开销
