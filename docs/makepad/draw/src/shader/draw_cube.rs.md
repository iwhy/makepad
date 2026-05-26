# `draw_cube.rs` — 3D 立方体着色器（DrawCube）

本文件实现了 Makepad 的基础 3D 立方体着色器，支持顶点光照、透视投影和纹理映射。是 draw crate 中唯一的纯 3D 原语着色器（DrawPbr 另计为完整的 PBR 材质系统）。

## `mod.draw.DrawCube` — 立方体着色器

### 注册

```rust
mod.draw.DrawCube = mod.std.set_type_default() do #(DrawCube::script_shader(vm)){...}
```

### 图形管线

#### Vertex Shader — `fn vertex()`

1. **局部坐标变换**：
   - 接收顶点位置 `geom_pos`（来自顶点缓冲的局部坐标，范围为 `[-0.5, 0.5]^3`）
   - 通过 `model_matrix` 将局部坐标变换到世界空间：`world_pos = model_matrix * vec4(geom_pos, 1.0)`
   - 通过 view-projection 矩阵变换到裁剪空间：`gl_Position = camera_projection * camera_view * world_pos`

2. **法线变换**：
   - 将顶点法线 `geom_normal` 通过法线矩阵（model 矩阵的逆转置）变换到世界空间
   - 法线用于后续的光照计算

3. **光照计算（逐顶点）**：
   - 使用 `light_direction`（归一化世界空间光照方向）和 `light_color`（光照颜色和强度）
   - 漫反射：`diffuse = max(dot(normal, light_dir), 0.0) * light_color.rgb`
   - 环境光：`ambient = ambient_color.rgb * ambient_intensity`
   - 顶点颜色 = `ambient + diffuse`
   - 逐顶点光照是性能优化（相比逐像素），代价是光照精度略低

4. **纹理坐标传递**：
   - `varying_uv = geom_uv`：将 UV 坐标传递到片段着色器

5. **输出组装**：
   - `varying_color = vertex_color * diffuse_ambient_term`
   - `varying_uv` 传递纹理坐标

### Fragment Shader — `fn pixel()`

1. **纹理采样**（如果启用）：
   - 使用 `varying_uv` 从 `cube_texture` (sampler2D) 采样
   - 纹理颜色与 `varying_color` 相乘：`final_color = texture_color * varying_color`

2. **无纹理渲染**（如果纹理未绑定）：
   - 直接使用 `varying_color`（包含光照计算结果）

3. **输出**：
   - 简单颜色输出到帧缓冲区
   - 支持 alpha 透明

### Uniforms

- **`cube_texture: sampler2D`**：立方体贴图纹理（可绑定普通 2D 纹理）
- **`model_matrix: mat4`**：模型变换矩阵
- **`camera_view: mat4`**：视图变换矩阵
- **`camera_projection: mat4`**：投影变换矩阵
- **`light_direction: vec3`**：归一化光照方向（世界空间）
- **`light_color: vec3`**：光照颜色和强度
- **`ambient_color: vec3`**：环境光颜色
- **`ambient_intensity: float`**：环境光强度

### Vertex Buffer (Geom)

- **`geom_pos: vec3`**：顶点位置（局部坐标，范围 `[-0.5, 0.5]`）
- **`geom_normal: vec3`**：顶点法线
- **`geom_uv: vec2`**：纹理坐标

### 使用场景

- 简单的 3D UI 元素（旋转立方体按钮、图标容器等）
- 3D 场景中的基础几何体
- 调试可视化

### 局限性

- 使用逐顶点光照，不适用于需要精细光照细节的场景
- 不支持法线贴图、镜面高光等高级光照效果
- 没有阴影映射
- 对于复杂 3D 场景，应使用 DrawPbr
