# `headless/shader_runtime_preamble.rs` — 运行时着色器前置代码

## 概述

本文件通过 `include_str!` 宏嵌入到每个 JIT 编译的着色器模块中。它提供了完整的 Rust 着色器运行时环境，包括向量类型、矩阵类型、纹理采样、SDF 渲染、数学内置函数等，实现了与 GPU 着色器语言（GLSL/Metal/HLSL）等价的功能集。

**特点**: 零外部依赖，纯标准库实现。

---

## 向量类型

### `Vec2f`, `Vec3f`, `Vec4f`

`#[repr(C)]` 结构体，分别包含 2/3/4 个 `f32` 字段：

| 类型 | 字段 |
|------|------|
| `Vec2f` | `x, y` |
| `Vec3f` | `x, y, z` |
| `Vec4f` | `x, y, z, w` |

类型别名：`vec2f`, `vec3f`, `vec4f`, `mat4x4f`。

### 构造函数

```rust
pub const fn vec2(x: f32, y: f32) -> Vec2f
pub const fn vec3(x: f32, y: f32, z: f32) -> Vec3f
pub const fn vec4(x: f32, y: f32, z: f32, w: f32) -> Vec4f
```

同时提供 `vec2f`, `vec3f`, `vec4f` 别名以适应着色器编译器生成的代码。

### 运算符重载

对三个向量类型提供了完整的运算符：

| 运算符 | 单目 | 向量-向量 | 向量-标量 |
|--------|------|-----------|-----------|
| `+` | | `impl Add` | `impl Add<f32>` |
| `-` | `impl Neg` | `impl Sub` | `impl Sub<f32>` |
| `*` | | `impl Mul` | `impl Mul<f32>` + 标量左侧 `impl Mul<VecNf> for f32` |
| `/` | | `impl Div` | `impl Div<f32>` |

复合赋值运算符（`AddAssign`, `SubAssign`, `MulAssign<f32>`, `DivAssign<f32>`）：

### Swizzle 方法

支持丰富的分量重排：

| 向量 | swizzle 示例 |
|------|-------------|
| `Vec2f` | `.xx()`, `.xy()`, `.yx()`, `.yy()`, `.xxx()`, `.xyx()`, `.xxxx()`, `.xyxy()` 等 |
| `Vec3f` | `.xx()`, `.xy()`, `.xz()`, `.xyz()`, `.xzy()`, `.yxz()`, `.zyx()`, `.xxxx()`, `.xyzz()`, `.xyzx()` 等 |
| `Vec4f` | `.xx()`, `.xy()`, `.xz()`, `.xw()`, `.rgb()`, `.xyz()`, `.xyzw()`, `.rgba()`, `.wzyx()`, `.zyxw()` 等 |

每个向量类型还有 `.mix(other, t)` 方法用于线性插值。

### `Vec2f` 特有：`.atan2()` 方法

返回 `self.y.atan2(self.x)`。

---

## `Mat4f` — 4×4 矩阵

`[repr(C)]` 结构体，包含 `[f32; 16]` 数组（列主序）。

| 方法 | 说明 |
|------|------|
| `identity()` | 返回 4×4 单位矩阵 |
| `invert()` | 使用余子式展开法求逆矩阵，行列式为零时返回单位矩阵 |

**`Mat4f × Vec4f` 乘法**: 使用标准的矩阵-向量乘法，将矩阵列与向量分量相乘后求和。

---

## `Texture2D` — 运行时纹理采样

`#[repr(C)]` 的纯 POD 类型，可以直接嵌入 `RenderCx` 结构体：

| 字段 | 说明 |
|------|------|
| `data_ptr: usize` | 纹理数据指针（作为整数存储，零表示无数据）|
| `data_len: usize` | f32 元素数量 |
| `width: usize` | 纹理宽度 |
| `height: usize` | 纹理高度 |

### 采样方法

**`sample_2d(coord: Vec2f) -> Vec4f`**:
1. 使用 `rem_euclid(1.0)` 实现重复寻址模式
2. 调用 `sample_face_from_data` 执行双线性插值

**`sample_cube(dir: Vec3f) -> Vec4f`**:
1. 归一化方向向量
2. 根据最大轴分量选择 6 个面（+X/-X/+Y/-Y/+Z/-Z）
3. 计算面内 UV 坐标并映射到 [0,1]
4. 调用 `sample_face_from_data` 执行采样

**`sample_face_from_data(data, face, coord) -> Vec4f`**:
1. 双线性插值（Bilinear filtering）
2. `Clamp-to-edge` 寻址模式
3. 计算：
   ```
   fx = u * width - 0.5, fy = v * height - 0.5
   x0 = floor(fx), y0 = floor(fy)
   tx = fx - x0, ty = fy - y0
   c00..c11 → 四个邻近纹素
   result = lerp(lerp(c00, c10, tx), lerp(c01, c11, tx), ty)
   ```

**`sample()` 泛型方法**: 通过 `TextureSampleCoord` trait 支持 `Vec2f`（2D 纹理）和 `Vec3f`（立方体贴图）的统一接口。

---

## 着色器数学内置函数

### 单参数函数

| 函数 | 等价于 |
|------|--------|
| `sign(x)` | `x.signum()` |
| `sqrt(x)` | `x.sqrt()` |
| `inverse_sqrt(x)` | `1.0 / x.sqrt()` |
| `inverse(m)` | `m.invert()` |
| `fract(x)` | `x - floor(x)` |
| `floor/ceil/round(x)` | Rust `f32` 方法 |
| `sin/cos/tan/asin/acos/atan(x)` | 标准三角函数 |
| `exp/exp2/log/log2(x)` | 指数/对数函数 |
| `pow(x, y)` | `x.powf(y)` |
| `abs(x)` | `x.abs()` |

### 双参数/三参数函数

| 函数 | 说明 |
|------|------|
| `step(edge, x)` | `x < edge ? 0 : 1` |
| `smoothstep(e0, e1, x)` | 三次 Hermite 插值 |
| `mix(a, b, t)` | `a + (b - a) * t`（泛型 trait 实现）|
| `clamp(x, lo, hi)` | `x.max(lo).min(hi)` |
| `modf(x, y)` | `x - y * floor(x / y)` |
| `atan2(y, x)` | `y.atan2(x)` |

### 向量函数

所有数学函数都有 `_2f`, `_3f`, `_4f` 后缀的向量版本：

| 标量 | Vec2f 版本 | Vec3f 版本 | Vec4f 版本 |
|------|-----------|-----------|-----------|
| `floor` | `floor_2f` | `floor_3f` | `floor_4f` |
| `ceil` | `ceil_2f` | `ceil_3f` | `ceil_4f` |
| `fract` | `fract_2f` | `fract_3f` | `fract_4f` |
| `round` | `round_2f` | `round_3f` | `round_4f` |
| `sign` | `sign_2f` | `sign_3f` | `sign_4f` |
| `sqrt` | `sqrt_2f` | `sqrt_3f` | `sqrt_4f` |
| `sin` | `sin_2f` | `sin_3f` | `sin_4f` |
| `cos` | `cos_2f` | `cos_3f` | `cos_4f` |
| `step` | `step_2f` | `step_3f` | `step_4f` |
| `smoothstep` | `smoothstep_2f` | `smoothstep_3f` | `smoothstep_4f` |
| `clamp` | `clamp_2f` | `clamp_3f` | `clamp_4f` |
| `max` | `max_2f` | `max_3f` | `max_4f` |
| `min` | `min_2f` | `min_3f` | `min_4f` |
| `abs` | `abs_2f` | `abs_3f` | `abs_4f` |

### 几何函数

| 函数 | 说明 |
|------|------|
| `distance_2f(a, b)` | `length(a - b)` |
| `length_2f(v)` | `sqrt(v.x² + v.y²)` |
| `dot_2f(a, b)` | `a.x*b.x + a.y*b.y` |
| `normalize_2f(v)` | `v / length(v)`（零向量返回零向量）|
| `dot_3f(a, b)` | 三维点积 |
| `length_3f(v)` | 三维长度 |
| `normalize_3f(v)` | 三维归一化 |
| `cross(a, b)` | 三维叉积 |
| `dot_4f(a, b)` | 四维点积 |
| `length_4f(v)` | 四维长度 |

### `mix()` — 泛型混合函数

```rust
pub trait Mix<T> {
    type Output;
    fn mix_impl(self, b: Self, t: T) -> Self::Output;
}
```

为 `f32`, `Vec2f`, `Vec3f`, `Vec4f` 实现，支持标量和向量权重。

### 颜色工具

`hsv_to_rgb(hsv: Vec4f) -> Vec4f` — 标准 HSV→RGB 转换。

---

## `Sdf2d` — 二维有符号距离场渲染器

实现基于 SDF（Signed Distance Field）的 2D 矢量图形渲染：

### 状态字段

| 字段 | 说明 |
|------|------|
| `pos` | 当前像素位置 |
| `result` | 累积渲染结果（预乘 Alpha）|
| `last_pos / start_pos` | 路径绘制状态 |
| `shape` | 当前 SDF 距离 |
| `clip` | 裁剪距离 |
| `has_clip` | 是否启用裁剪 |
| `blur / aa` | 模糊和抗锯齿参数 |
| `scale_factor` | 缩放因子 |
| `dist` | 距离值 |

### 初始化

```rust
Sdf2d::viewport_f2(pos)  // 从像素位置创建
```

### 图形基元

| 方法 | 说明 |
|------|------|
| `circle_f3(x, y, r)` | 圆形 SDF |
| `box_f4(x, y, w, h, r)` | 圆角矩形 SDF |
| `rect_f4(x, y, w, h)` | 矩形 SDF |

### 路径绘制

| 方法 | 说明 |
|------|------|
| `move_to(x, y)` | 移动到起点 |
| `line_to(x, y)` | 线段 SDF（点到线段距离）|
| `close_path()` | 闭合路径（回到起点）|

### 布尔运算

| 方法 | 说明 |
|------|------|
| `union()` | `min(shape, old_shape)` |
| `intersect()` | `max(shape, old_shape)` |
| `subtract()` | `max(-shape, old_shape)` |

### 渲染输出

| 方法 | 说明 |
|------|------|
| `fill_keep(color)` | 使用 SDF 值填充，保持路径状态 |
| `fill(color)` | 填充后重置路径 |
| `stroke_keep(color, width)` | 描边（使用 `|d| - width/2`），保持路径 |
| `stroke(color, width)` | 描边后重置 |
| `glow_keep(color, width)` | 发光效果（高斯衰减），保持路径 |
| `glow(color, width)` | 发光后重置 |

所有渲染输出使用预乘 Alpha 混合叠加到 `result` 上：
```rust
let alpha = (-d / aa + 0.5).clamp(0, 1);   // 抗锯齿透明度
let premul = vec4(color.r * color.a * alpha, ...);
result = premul + result * (1 - premul.a);   // SrcOver 混合
```
