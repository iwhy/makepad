# `draw_text_3d.rs` — 3D 文本着色器（DrawText3d）

本文件实现了 3D 空间中的文本渲染着色器，扩展 `DrawRotatedText` 使其支持在 3D 场景中渲染文本。提供广告牌模式（始终面向相机）和面向模式（沿指定方向）两种空间定位方式。

## `mod.draw.DrawText3d` — 3D 文本着色器

### 注册

```rust
mod.draw.DrawText3d = mod.std.set_type_default() do #(DrawText3d::script_shader(vm)){...}
```

### 继承关系

```
DrawText（基础文本渲染）
  └─ DrawRotatedText（+ 2D 旋转矩阵）
       └─ DrawText3d（+ 3D 投影 / 广告牌）
```

使用 `..mod.draw.DrawRotatedText` 继承所有文本渲染功能和 2D 旋转能力。

### 新增功能

- 3D 空间定位（使用 3D 变换矩阵）
- **广告牌模式**：文本始终面向相机，用于标签、名称标注
- **面向模式**：文本沿指定方向定位，不自动旋转朝向相机
- 3D 场景中的文本缩放（保持屏幕大小一致或随距离缩放）

### Uniforms

- **`text_model_matrix: mat4`**：文本的 3D 模型变换矩阵（位置、旋转、缩放）
- **`text_camera_view: mat4`**：视图矩阵
- **`text_camera_projection: mat4`**：投影矩阵
- **`billboard_mode: float`**：广告牌模式开关：
  - `0.0`：面向模式（3D 定位但不旋转面向相机）
  - `1.0`：广告牌模式（始终面向相机）
- **`keep_screen_size: float`**：屏幕尺寸保持开关：
  - `0.0`：随距离缩放（近大远小）
  - `1.0`：保持屏幕像素尺寸不变

### Vertex Shader — `fn vertex()`

1. **模式判断**：
   - 检查 `billboard_mode` 确定使用哪种空间变换

2. **广告牌模式（billboard_mode = 1.0）**：
   - 计算文本中心的世界空间位置
   - 构建广告牌旋转矩阵：
     - 从 `text_camera_view` 提取相机的右向量和上向量
     - 构建旋转矩阵 = `[camera_right, camera_up, camera_forward]`
   - 文本始终以相同的面向角度朝向相机
   - 文本的局部缩放保持

3. **面向模式（billboard_mode = 0.0）**：
   - 使用 `text_model_matrix` 进行完整 3D 变换
   - 文本的旋转和缩放由模型矩阵完全控制
   - 透视投影生效（近大远小）

4. **屏幕尺寸保持**：
   - `keep_screen_size = 1.0` 时，计算从相机到文本的距离
   - 根据距离反向调整文本缩放，使屏幕投影尺寸恒定
   - 公式：`scale = reference_distance / actual_distance`

5. **标准管线**：
   - 将顶点变换到裁剪空间：`gl_Position = projection * view * model * vec4(vertex, 1.0)`
   - 传递 UV、颜色、Slug 参数到片段着色器

### Fragment Shader — `fn pixel()`

- 与 `DrawRotatedText` 的像素着色器完全相同
- 所有 3D 变换在顶点着色器中完成
- 片段着色器专注文本渲染（SDF 计算、抗锯齿、纹理采样、颜色输出）

### 3D 空间定位说明

- **坐标系统**：文本在局部坐标中排版（原点位于文本左上角或中心），通过模型矩阵变换到世界空间
- **深度测试**：启用深度测试，文本在 3D 场景中正确遮挡
- **透明度排序**：对于半透明文本，使用 painter's algorithm 从后往前渲染
- **裁剪**：支持 3D 视锥体裁剪

### 使用场景

- 3D 场景中的标签系统（人物名称标注、物体标签）
- 增强现实 / 虚拟现实界面中的文字
- 3D 数据可视化中的标注
- 游戏中的对话气泡、伤害数字
- 3D 交互 UI（如 VR 菜单）

### 与 DrawRotatedText 的关系

- `DrawText3d` 在 `DrawRotatedText` 的 2D 旋转基础上增加完整的 3D 变换管线
- 核心文本渲染逻辑完全复用 `DrawRotatedText` 的片段着色器
- 新功能全部在顶点着色器中实现（坐标变换、广告牌计算）
- 可通过统一着色器实现从 2D UI 到 3D 场景的无缝过渡
