# `draw_pbr.rs` — PBR 着色器（DrawPbr）

本文件是 draw crate 中最大的着色器文件（约 85KB），实现了完整的 PBR（Physically Based Rendering）材质系统。支持 glTF 2.0 材质标准、图像基础光照（IBL）、环境贴图、阴影映射、网格缓存和材质属性系统。

## `mod.draw.DrawPbr` — 标准 PBR 着色器

### 注册

```rust
mod.draw.DrawPbr = mod.std.set_type_default() do #(DrawPbr::script_shader(vm)){...}
```

### 材质模型

使用 Cook-Torrance BRDF 微表面模型：
- **漫反射**：Lambertian 漫反射 `diffuse = base_color / PI`
- **镜面反射**：使用 GGX/Trowbridge-Reitz 法线分布函数（NDF）
- **几何遮挡**：Smith 几何函数（GGX 相关）
- **菲涅尔效应**：Schlick 近似

### Fragment Shader — `fn pixel()` 执行流程

1. **输入解码**：
   - `varying_uv`：纹理坐标
   - `varying_normal`：世界空间法线
   - `varying_pos`：世界空间位置
   - `varying_color`：逐顶点颜色

2. **材质参数组装**：
   - 从材质属性缓冲中读取 PBR 参数
   - **base_color**：基础颜色（来自 `base_color_texture` 或 uniform）
   - **metallic**：金属度（0 = 电介质，1 = 金属）
   - **roughness**：粗糙度（0 = 完美镜面，1 = 完全粗糙）
   - **normal_map**：法线贴图采样，扰动表面法线
   - **occlusion**：环境光遮蔽（来自 `occlusion_texture`）
   - **emissive**：自发光颜色和强度

3. **法线处理**：
   - 从 `normal_texture` 采样法线贴图
   - 使用 TBN 矩阵将切线空间法线变换到世界空间
   - 如果无法线贴图，使用顶点法线

4. **光照计算**：
   - **直接光照**：对所有有效光源循环计算
     - 计算光照方向 `L`、视角方向 `V`、半角向量 `H`
     - 计算 NDF（GGX）：`D = alpha^2 / (PI * (cos_theta_h^2 * (alpha^2 - 1) + 1)^2)`
     - 计算几何遮挡（Smith GGX）：`G = G1(L) * G1(V)`
     - 计算菲涅尔（Schlick）：`F = F0 + (1 - F0) * (1 - cos_theta)^5`
     - 输出 = `(diffuse + specular * D * G * F / (4 * cos_theta_l * cos_theta_v)) * light_color * shadow_term`
   - **环境光照（IBL）**：
     - 漫反射 IBL：采样 irradiance 贴图 `irradiance_map`
     - 镜面 IBL：采样 pre-filtered 环境贴图 `radiance_map` + BRDF LUT
     - IBL 强度通过 `ibl_intensity` 控制

5. **阴影映射**：
   - 从光源视角的深度贴图 `shadow_map` 采样
   - 使用 PCF（Percentage Closer Filtering）进行软阴影
   - 支持级联阴影映射（CSM）的参数
   - `shadow_bias` 用于阴影痤疮修正

6. **最终输出**：
   - `final_color = direct_lighting + indirect_lighting + emissive`
   - 应用 `occlusion` 减弱环境光照
   - Tone mapping（如果启用）
   - Gamma 校正
   - HDR 输出

### Uniforms

- **纹理**：`base_color_texture`、`normal_texture`、`metallic_roughness_texture`、`occlusion_texture`、`emissive_texture`
- **IBL**：`irradiance_map`、`radiance_map`、`brdf_lut`
- **阴影**：`shadow_map`、`shadow_matrix`、`shadow_bias`、`cascade_distances`
- **PBR 参数**：`base_color_factor`、`metallic_factor`、`roughness_factor`、`emissive_factor`
- **场景**：`camera_position`、`light_direction`、`light_color`、`light_intensity`

### Vertex Buffer (Geom)

- `geom_pos`：顶点位置
- `geom_normal`：顶点法线
- `geom_tangent`：顶点切线（用于法线贴图 TBN 矩阵）
- `geom_uv`：纹理坐标
- `geom_color`：顶点颜色

## `mod.draw.DrawPbrRefractive` — 折射 PBR 着色器

扩展 `DrawPbr`，增加透明材质的折射效果：
- 使用 `refraction_index` 控制折射率（IOR）
- 采样背景场景进行折射扭曲
- 支持薄片透明（thin-wall）和实体透明（solid）模式
- 结合菲涅尔效应，在掠射角切换到反射

## `mod.draw.DrawPbrMesh` — PBR 网格管理

### 功能

- **网格缓存**：对加载的 glTF 网格进行 GPU 缓冲管理
- **Mesh 属性**：位置、法线、切线、UV、颜色、骨骼权重等
- **Sub-mesh 支持**：每个网格可以包含多个材质区域
- **LOD 支持**：多级细节网格

### 网格加载管线

1. 从 glTF 文件解析网格数据
2. 创建 GPU 顶点缓冲和索引缓冲
3. 生成网格包围盒供裁剪使用
4. 缓存网格引用供后续帧复用

## `mod.draw.PbrMaterial` — PBR 材质

### 注册

```rust
mod.draw.PbrMaterial = mod.std.set_type_default() do #(PbrMaterial::script_component(vm)){...}
```

基于 glTF 2.0 材质规范：

- **基础属性**：`base_color`（vec4）、`metallic`（float）、`roughness`（float）
- **纹理引用**：`base_color_texture`、`normal_texture`、`metallic_roughness_texture`、`occlusion_texture`、`emissive_texture`
- **纹理变换**：每个纹理支持独立的 `offset`/`scale`/`rotation`
- **高级**：`alpha_mode`（OPAQUE/MASK/BLEND）、`alpha_cutoff`、`double_sided`
- **自发光**：`emissive_factor`（vec3）、`emissive_strength`

### 材质系统

- 材质可以在运行时修改参数
- 支持材质实例（MaterialInstance）实现 per-object 参数覆盖
- 纹理通过 `TextureCache` 引用，自动管理加载和卸载

## `mod.draw.PbrLight` — PBR 光源

### 注册

```rust
mod.draw.PbrLight = mod.std.set_type_default() do #(PbrLight::script_component(vm)){...}
```

- **方向光**：`direction`、`color`、`intensity`
- **点光源**：`position`、`color`、`intensity`、`range`、`decay`
- **聚光灯**：`position`、`direction`、`color`、`intensity`、`inner_cone_angle`、`outer_cone_angle`
- 支持最多 4 个有效光源同时作用（性能限制）
- 阴影映射仅对主方向光启用

## 渲染管线

1. **深度预通道**（可选）：生成深度缓冲供阴影映射使用
2. **阴影映射通道**：从光源视角渲染深度到 `shadow_map`
3. **主渲染通道**：执行完整的 PBR 着色器
4. **透明通道**：对 `alpha_mode == BLEND` 的材质进行排序后渲染

## 性能优化

- 使用 uniform 缓冲批量更新材质参数
- 支持 GPU instancing（相同网格+不同变换）
- 视锥体裁剪剔除不可见网格
- 遮挡剔除（使用上一帧的深度缓冲）
