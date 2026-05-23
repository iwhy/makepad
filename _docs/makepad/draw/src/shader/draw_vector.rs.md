# `draw_vector.rs` — 矢量路径着色器（DrawVector）

本文件实现了 Makepad 的矢量路径渲染引擎，支持任意 2D 路径的填充和描边。使用基于渐变纹理的 SDF 光栅化技术，支持线性渐变、径向渐变和纹理填充。

## `mod.draw.DrawVector` — 矢量路径着色器

### 注册

```rust
mod.draw.DrawVector = mod.std.set_type_default() do #(DrawVector::script_shader(vm)){...}
```

### 核心功能

- 任意 2D 路径的填充（非零/奇偶填充规则）
- 描边（可变宽度、dash 样式）
- 渐变填充（线性渐变、径向渐变）
- 纹理填充（UV 映射）
- 抗锯齿（基于 SDF）

### Uniforms

- **`gradient_texture: sampler2D`**：渐变色纹理（1D 或 2D），存储渐变颜色查找表
- **`gradient_uv_matrix: mat4`**：渐变 UV 变换矩阵，将路径空间坐标映射到渐变纹理 UV
- **`vector_line_texture: sampler1D`**：线纹理（dash 纹理），控制描边的虚线和实线模式
- **`fill_mode: vec4`**：填充模式参数：
  - `x`：填充类型（0 纯色 / 1 线性渐变 / 2 径向渐变 / 3 纹理）
  - `y`：渐变起始比例
  - `z`：渐变终止比例
  - `w`：填充规则（0 非零 / 1 奇偶）
- **`fill_color: vec4`**：纯色填充颜色
- **`stroke_color: vec4`**：描边颜色
- **`stroke_width: float`**：描边宽度

### Vertex Output / Fragment Input

- **`varying_color: vec4`**：插值颜色
- **`varying_path_id: float`**：当前路径的 ID（用于在 SDF 中查找路径数据）
- **`varying_bounds: vec4`**：路径包围盒

### Fragment Shader — `fn pixel()` 执行流程

1. **SDF 距离计算**：
   - 使用 `varying_path_id` 从路径数据缓冲中提取当前像素附近的路径段
   - 计算像素到所有路径段的有符号距离
   - 使用 `fill_rule`（非零/奇偶）确定填充区域

2. **渐变计算**：
   - **线性渐变**：使用 `gradient_uv_matrix` 将像素坐标投影到渐变轴，计算线性插值位置 `t`，从 `gradient_texture` 采样颜色
   - **径向渐变**：计算像素到渐变中心的距离，归一化后采样渐变纹理
   - 支持渐变重复、镜像、钳制模式

3. **纹理填充**：
   - 使用预计算的 UV 矩阵将像素坐标映射到纹理 UV
   - 采样纹理颜色与填充颜色混合

4. **描边渲染**：
   - 如果启用描边，额外计算像素到路径轮廓的距离
   - 使用 `stroke_width` 确定描边区域
   - 如果启用 dash，使用 `vector_line_texture` 采样 dash 模式
   - 描边颜色 = `stroke_color * coverage`

5. **抗锯齿**：
   - 在路径边缘使用 `smoothstep` 进行抗锯齿过渡
   - 过渡宽度 = 1 像素（使用 `aa_viewport` 计算）

### 渐变纹理详细说明

渐变纹理是一个 1D 纹理（宽度为 `GRADIENT_TEXTURE_SIZE`），存储了渐变的颜色查找表：

- **线性渐变创建**：
  1. 在 CPU 端计算渐变轴方向向量
  2. 将路径包围盒内的每个点投影到渐变轴
  3. 生成 1D 纹理数据，每个纹素对应一个颜色值
  4. 上传到 `gradient_texture`

- **径向渐变创建**：
  1. 计算中心点和半径
  2. 对每个像素计算到中心的距离 `d`
  3. `t = d / radius`，使用 `t` 采样渐变纹理
  4. 支持焦点偏移（focal point）

### 路径数据组织

路径数据以紧凑格式存储在 GPU 缓冲中：
- 每个路径段使用固定大小的数据结构
- 直线段使用 2 个顶点（起点 + 终点）
- 二次贝塞尔使用 3 个顶点（起点 + 控制点 + 终点）
- 三次贝塞尔使用 4 个顶点（起点 + 控制点1 + 控制点2 + 终点）
- 路径段数据通过 `varying_path_id` 索引

### 与 DrawQuad 的关系

- `DrawVector` 提供通用路径渲染（任意形状）
- `DrawQuad` 是 `DrawVector` 的矩形特化版本（优化路径）
- `DrawQuad` 在矩形渲染性能上优于 `DrawVector`
- `DrawVector` 用于复杂形状（图标、SVG、自定义路径）

### 与其他着色器的关系

- `draw_svg.rs` 中的 `DrawSvg` 通过 `..mod.draw.DrawVector` 继承并添加 SVG 颜色覆盖
- `draw_quad.rs` 中的 `DrawQuad` 可以视为 `DrawVector` 的矩形优化版
