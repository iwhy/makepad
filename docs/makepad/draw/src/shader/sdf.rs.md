# `sdf.rs` — SDF 绘图原语与着色器工具函数

本文件定义了一组底层 GPU 绘图原语和工具函数，注册在 `mod.draw` 命名空间下，供其他所有 draw 着色器调用。包括 `GaussShadow`（高斯模糊阴影）、`Math`（数学工具）、`Pal`（调色板）、`Sdf2d`（2D SDF 渲染结构体）以及 `Sdf2dExt`（扩展方法）。

## `mod.draw.Math` — 数学工具

### 注册

```rust
mod.draw.Math = mod.std.set_type_default(){}
```

无状态函数集合，通过 `mod.draw.Math` 命名空间访问。

### 函数

- **`circularize(v: float) -> float`**：将线性值 `v` 转换为圆形插值曲线，公式为 `1.0 - sqrt(1.0 - v*v)`。用于缓动动画，从慢到快的自然加速效果。

- **`average(v: vec2) -> float`**：计算 `v.x` 和 `v.y` 的平均值，返回 `(v.x + v.y) * 0.5`。

- **`almost_equal(a: float, b: float) -> float`**：检查两个浮点数是否近似相等，返回 `abs(a - b) < 0.0001` 的布尔值（1.0/0.0）。用于避免浮点精度问题导致的渲染瑕疵。

- **`lerp(a: float, b: float, t: float) -> float`**：标准线性插值，返回 `a + (b - a) * t`。当 `t=0` 时返回 `a`，`t=1` 时返回 `b`。

- **`inv_lerp(a: float, b: float, v: float) -> float`**：`lerp` 的逆运算，计算 `v` 在 `[a, b]` 区间内的归一化位置 `(v - a) / (b - a)`。当 `v=a` 时返回 `0`，`v=b` 时返回 `1`。

- **`smooth_in_out(t: float) -> float`**：SmoothStep 缓动函数，实现平滑的加速-减速曲线。返回 `t*t*(3.0 - 2.0*t)`，在 `t=0` 和 `t=1` 处的导数为零，确保运动开始和结束时的平滑过渡。

- **`pulsate(t: float, period: float, offset: float) -> float`**：周期性脉冲函数。通过 `sin(t/period + offset)` 生成正弦波，再映射到 `[0,1]` 范围。用于呼吸动画、闪烁效果等周期性视觉反馈。

- **`bilinear_sample(s: float, t: float, c00: vec4, c01: vec4, c10: vec4, c11: vec4) -> vec4`**：双线性插值采样。在四个角点 `c00`(top-left), `c01`(top-right), `c10`(bottom-left), `c11`(bottom-right) 之间，以 `s`(水平)和 `t`(垂直)为权重进行插值。先水平插值两个边缘，再垂直插值结果。用于纹理采样、渐变计算等。

- **`rad_to_deg(r: float) -> float`**：弧度转角度，返回 `r * 180.0 / Pi`。

- **`deg_to_rad(d: float) -> float`**：角度转弧度，返回 `d * Pi / 180.0`。

- **`rotate_point(point: vec2, angle: float) -> vec2`**：2D 点绕原点旋转。使用 `cos(angle)` 和 `sin(angle)` 构建旋转矩阵，计算旋转后的坐标。用于文本旋转、UI 元素变换等。

## `mod.draw.Pal` — 调色板工具

### 注册

```rust
mod.draw.Pal = mod.std.set_type_default(){}
```

### 函数

- **`cosine_palette(t: float, a: vec3, b: vec3, c: vec3, d: vec3) -> vec3`**：基于余弦的调色板生成函数。通过 `a + b * cos(6.28318 * (c * t + d))` 公式，用四个向量参数控制颜色曲线的幅度、频率和相位。`a` 控制颜色偏移（亮度），`b` 控制幅度（对比度），`c` 控制频率（色彩变化速度），`d` 控制相位（色调偏移）。通过调整四个参数，可以从同一公式生成无限多种渐变色方案。用于数据可视化、程序化纹理生成等场景。

## `mod.draw.GaussShadow` — 高斯阴影（圆角矩形阴影）

### 注册

```rust
mod.draw.GaussShadow = mod.std.set_type_default() do #(GaussShadow::script_shader(vm)){...}
```

### Uniforms

- **`inv_shadow_radius`**：阴影半径的倒数，用于在着色器中将屏幕空间距离转换为标准化坐标。
- **`shadow_rect`**：阴影矩形的四边位置 `(left, top, right, bottom)`，定义阴影的边界范围。
- **`shadow_gradient`**：四向渐变纹素 `(left, top, right, bottom)`，存储每个方向上的高斯卷积纹素，用于沿矩形四边进行模糊。
- **`shadow_color`**：阴影颜色 `(r, g, b, a)`。
- **`shadow_radius`**：阴影圆角半径，定义阴影的圆角大小。

### 函数

- **`set_rect_from_points(cx: &mut Cx, pos: vec2, size: vec2, radius: float)`**：从位置和大小设置阴影矩形参数。计算矩形四边位置、阴影半径和 `inv_shadow_radius`，并调用 `set_rect_gradient` 计算四向高斯纹素。

- **`set_rect_gradient(cx: &mut Cx, shadow_gradient: &mut vec4)`**：计算四向高斯纹素。对阴影矩形的四条边分别计算高斯权重纹素，具体逻辑：
  1. 对每条边，在 `[0, 2*radius]` 范围内采样多个点
  2. 每个采样点计算高斯权重 `exp(-x*x)`，其中 `x` 是采样点到矩形边的距离
  3. 将各边的纹素打包到 `vec4` 的对应通道：`x`=左边, `y`=上边, `z`=右边, `w`=下边
  4. 纹素用于在像素着色器中沿对应方向进行一维高斯模糊

- **`shadow_fn(uv: vec2) -> float`**：核心 SDF 阴影函数。计算像素位置相对于阴影矩形四边的距离，应用高斯模糊权重：
  1. 计算像素到四边的归一化距离 `d = (uv - shadow_rect) * inv_shadow_radius`
  2. 对每条边的距离应用高斯权重 `exp(-d*d * 0.5)`
  3. 将四边的权重相加，根据距角落的距离混合圆角区域的权重
  4. 最终返回 `clamp(weight, 0, 1)` 作为阴影不透明度
  使用带符号距离的圆角矩形 SDF，在角落区域沿对角线方向混合相邻边的权重，确保圆角处的阴影过渡自然。

- **`shadow_fn_glyph(uv: vec2, glyph_bounds: vec4, glyph_sdf: float) -> float`**：字形阴影函数。在 `shadow_fn` 基础上增加字形边界约束：
  1. 先通过 `shadow_fn` 计算矩形阴影权重
  2. 使用 `glyph_bounds` 裁剪字形区域外的阴影
  3. 对字形内部区域，使用 `glyph_sdf`（字形的 SDF 值）来衰减阴影
  4. 在字形边缘附近平滑过渡，确保阴影不会溢出字形边界
  用于 DropShadow 特效中，使阴影精确匹配字形轮廓而不是整个文本框。

## `mod.draw.Sdf2d` — 2D SDF 渲染引擎（核心渲染原语）

### 注册

```rust
mod.draw.Sdf2d = mod.std.set_type_default() do #(Sdf2d::script_shader(vm)){...}
```

这是 Makepad 2D 渲染的核心结构体，提供完整的 2D 形状 SDF 渲染管线。使用 GPU 友好的 SDF 技术，避免传统三角剖分的开销。

### Uniforms

- **`aa_viewport`**：抗锯齿视口尺寸 `(width, height)`，用于将像素坐标映射到 SDF 空间并计算抗锯齿边缘宽度。
- **`fill_rule`**：填充规则，0=非零（NonZero），1=奇偶（EvenOdd），控制复杂自交路径的填充区域判定。
- **`gradient_texture`**：渐变纹理采样器，支持 `ClampToEdge`、线性过滤。用于路径内部的渐变填充。
- **`line_texture`**：一维材质纹理采样器，用于 dash 样式等线型纹理映射。

### 函数

- **`viewport(cx: &mut Cx, pos: vec2, size: vec2)`**：初始化 `Sdf2d` 的视口。设置 `aa_viewport` 为 `1.0/size`，计算 `pos`（视口原点偏移），重置内部状态（路径计数、包围盒、当前点、路径模式等）。

- **`absolute_move_to(cx: &mut Cx, p: vec2)`**：绝对坐标移动。将当前绘图点移动到 `p`，隐式结束当前子路径（如果正在构建）。用于路径起始和跳转。

- **`move_to(cx: &mut Cx, p: vec2)`**：相对坐标移动。以当前点为基准移动到 `p`。等价于 `absolute_move_to(current_point + p)`。

- **`line_to(cx: &mut Cx, p: vec2)`**：画直线段。从当前点到 `p` 画一条直线。更新包围盒，增加路径段计数器。

- **`absolute_line_to(cx: &mut Cx, p: vec2)`**：绝对坐标直线段。与 `line_to` 功能相同，但坐标是绝对坐标。

- **`cubic_to(cx: &mut Cx, cp1: vec2, cp2: vec2, p: vec2)`**：画三次贝塞尔曲线。使用两个控制点 `cp1`、`cp2` 和终点 `p`。更新包围盒时保守估计曲线覆盖的范围（包含控制点的包围盒）。更新路径段计数。

- **`absolute_cubic_to(cx: &mut Cx, cp1: vec2, cp2: vec2, p: vec2)`**：绝对坐标三次贝塞尔曲线。

- **`quad_to(cx: &mut Cx, cp: vec2, p: vec2)`**：画二次贝塞尔曲线。使用一个控制点 `cp` 和终点 `p`。更新包围盒。

- **`absolute_quad_to(cx: &mut Cx, cp: vec2, p: vec2)`**：绝对坐标二次贝塞尔曲线。

- **`close_path(cx: &mut Cx)`**：闭合当前子路径。从当前点画直线回到路径起点，标记路径结束。

- **`clear(cx: &mut Cx)`**：清除所有路径数据。重置路径编号、段计数、包围盒和填充规则。

- **`stroke(cx: &mut Cx)`**：设置绘制模式为描边。后续的 `fill` 调用将执行描边操作。

- **`stroke_width(cx: &mut Cx, w: float)`**：设置描边宽度。结合 `stroke` 使用，控制线条粗细。

- **`no_line_texture(cx: &mut Cx)`**：禁用线纹理。后续描边不使用 dash 等纹理映射。

- **`set_line_texture(cx: &mut Cx, ...)`**：设置线纹理。控制描边的 dash 样式、偏移等。

- **`fill(cx: &mut Cx, color: vec4)`**：核心填充函数。将所有累积的路径段渲染为填充或描边形状。实现逻辑：
  1. 为当前路径分配全局唯一的路径编号（自增 ID）
  2. 将颜色、路径编号、填充规则、描边属性等打包为顶点数据
  3. 根据包围盒生成覆盖形状的像素网格
  4. 每个像素产生一个片段，片段着色器通过路径编号访问 SDF 数据
  5. 使用 `sdf_pixel` 计算最终颜色

- **`stroke_fill(cx: &mut Cx, color: vec4)`**：同时填充和描边。先执行填充操作，再执行描边操作。

- **`fill_with_texture(cx: &mut Cx, uv_clip: vec4, texture: sampler2D, foreground_color: vec4, background_color: vec4)`**：带纹理的填充。除了路径渲染外，还使用 UV 坐标 `uv_clip` 采样纹理 `texture`，将纹理颜色与路径 SDF 值混合。用于在任意路径形状内显示纹理内容。

- **`path_bbox() -> vec4`**：计算当前路径的包围盒，返回 `(min_x, min_y, max_x, max_y)`。用于确定需要渲染的像素区域。

- **`sdf_pixel(pos: vec2, color: vec4, fill_id: float) -> vec4`**：SDF 像素着色器函数。对每个片段执行 SDF 计算：
  1. 从顶点数据中提取路径段的控制点和类型
  2. 使用 `signed_distance_to_path` 计算当前像素到路径的有符号距离
  3. 对距离应用 `smoothstep` 进行抗锯齿（宽度由 `aa_viewport` 控制）
  4. 根据填充规则确定像素是否在内部
  5. 如果启用渐变纹理，混合纹理颜色
  6. 返回最终颜色 = `color * coverage`

  用于并行渲染大量路径段，每个像素独立评估所有相关路径段。

- **`sdf_pixel_textured(...)`**：带纹理的 SDF 像素着色器。在 `sdf_pixel` 基础上增加纹理采样和混合逻辑。

- **`sdf_pixel_procedural(...)`**：过程化纹理 SDF 像素着色器。不采样纹理，而是使用程序化 UV 坐标计算颜色。

- **`sdf_path_bbox(pos: vec2) -> vec4`**：计算指定位置处路径的包围盒。在处理多个重叠路径时用于确定每个像素属于哪个路径。

- **`set_data(...)`**：直接设置 SDF 原始数据。用于从预计算的 SDF 数据初始化，避免重建路径。

- **`reset_and_clear(clear: vec2)`**：重置 SDF 状态并清除。根据 `clear` 参数决定是否清除路径数据。

### 内部 SDF 距离计算

`Sdf2d` 的片段着色器通过以下步骤计算像素到路径的距离：

1. **段分类**：将路径段分类为直线、二次曲线和三次曲线
2. **最近点计算**：对每段计算像素到曲线的最短距离
3. **符号判定**：使用 winding number 或 ray casting 确定点在路径内部还是外部
4. **距离合并**：从所有段中选择最短有符号距离
5. **抗锯齿**：使用 `aa_viewport` 的倒数作为边缘过渡宽度

## `mod.draw.Sdf2dExt` — SDF 扩展方法

### 注册

```rust
mod.draw.Sdf2dExt = mod.std.set_type_default(){}
```

### 函数

- **`box(all: vec4, corner_radius: float) -> Sdf2d`**：创建圆角矩形路径。`all=(x, y, w, h)`，自动生成四条边和四个圆角的路径段。圆角使用四分之一圆弧（二次或三次贝塞尔近似）。

- **`rounded_box(all: vec4, radius: vec4) -> Sdf2d`**：创建四角独立半径的圆角矩形。每个角可以有不同的圆角半径，用于实现更灵活的外观。

- **`polygon(all: vec4, ...) -> Sdf2d`**：创建正多边形路径。生成多边形顶点的直线段。

- **`ellipse(all: vec4) -> Sdf2d`**：创建椭圆路径。使用四条三次贝塞尔曲线近似椭圆弧。

- **`line(from: vec2, to: vec2) -> Sdf2d`**：创建单条直线路径。

- **`circle(center: vec2, radius: float) -> Sdf2d`**：创建圆形路径。生成全圆弧。

- **`ring(center: vec2, radius: float, thickness: float) -> Sdf2d`**：创建圆环路径。外圆和内圆组合成一个复合路径。

- **`arc(center: vec2, radius: float, ...) -> Sdf2d`**：创建圆弧路径。支持起始角度和扫描角度。

- **`path() -> Sdf2d`**：创建空路径。用于手动构建复杂形状。

- **`flat_box(all: vec4) -> Sdf2d`**：创建无圆角的纯矩形路径。用于简单场景，性能优于 `box`。

- **`flat_rounded_box(all: vec4, radius: vec4) -> Sdf2d`**：创建有圆角的扁平矩形路径。在不需要抗锯齿时使用。

### 调用方式

`Sdf2dExt` 的方法返回 `Sdf2d` 对象，支持链式调用。例如：

```rust
let sdf = Sdf2dExt.box(pos_size, 4.0)
sdf.fill(cx, color)
```
