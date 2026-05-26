# `shader_runtime.rs` 源码解读

**路径:** `libs/math/src/shader_runtime.rs`
**行数:** 742
**核心职责:** 提供 CPU 端着色器运行时环境：向量 swizzle 运算、纹理采样、颜色空间转换、二维 SDF 形状渲染

---

## 1. Swizzle 方法

### `impl Vec2f` — 二维向量 Swizzle

#### 2 分量 swizzle（返回 Vec2f）
- `xx, xy, yx, yy` — 从 Vec2f 的 x/y 重组成新的 Vec2f
- 模仿 GLSL 语法，如 `v.xx()` 产生 `(x, x)`

#### 3 分量 swizzle（返回 Vec3f）
- `xxx, xxy, xyx, xyy, yxx, yxy, yyx, yyy` — 全部 8 种 2 选 3 组合

#### 4 分量 swizzle（返回 Vec4f）
- `xxxx, xxyy, xyxy, yyxx` — 部分常见 4 分量组合

---

### `impl Vec3f` — 三维向量 Swizzle

#### 2 分量 swizzle（返回 Vec2f）
- `xx, xy, xz, yx, yy, yz, zx, zy, zz` — 全部 9 种 3 选 2 组合

#### 3 分量 swizzle（返回 Vec3f）
- `xxx, xxy, xxz, xyz, xzy, yxz, yzx, zxy, zyx, zzz` — 全部排列组合

#### 4 分量 swizzle（返回 Vec4f）
- `xxxx, xyzx, xyzz` — 常用 4 分量组合

#### `pub fn mix(&self, other: Vec3f, t: f32) -> Vec3f`
- 线性插值：`self + (other - self) * t`，每分量独立计算

---

### `impl Vec4f` — 四维向量 Swizzle

#### 2 分量 swizzle（返回 Vec2f）
- `xx, xz, xw, yx, yy, yz, yw, zx, zy, zz, wx, wy, wz, ww` — 全部 16 种 4 选 2 组合
- 注：`xy()` 和 `zw()` 已在 `math_f32.rs` 中定义

#### 3 分量 swizzle（返回 Vec3f）
- `xxx, xyz, xyw, xzy, yxz, yzw, zxy, zyx, zwx, wxy, wzx` — 常用三维组合
- `rgb()` — 语义别名，等价于 `xyz()`

#### 4 分量 swizzle（返回 Vec4f）
- `xyzw, xxxx, yyyy, zzzz, wwww, wzyx, zyxw` — 包括反转和广播
- `rgba()` — 语义别名，等价于 `xyzw()`

---

## 2. Vec2f 附加方法

### `pub fn mix(&self, other: Vec2f, t: f32) -> Vec2f`
- 线性插值

### `pub fn atan2(&self) -> f32`
- 向量的极角：`y.atan2(x)`，返回弧度

---

## 3. Vec4f 附加方法

### `pub fn mix(&self, other: Vec4f, t: f32) -> Vec4f`
- 四维线性插值

---

## 4. Texture2D — CPU 端纹理采样

### 类型定义
- **[width]**: `usize` — 纹理宽度（像素）
- **[height]**: `usize` — 纹理高度（像素）
- **[data]**: `Vec<f32>` — RGBA 像素数据，行主序存储，每像素 4 个 f32

### `impl Texture2D`

#### `pub fn new() -> Self`
- 创建空纹理（宽高为 0，空数据）

#### `pub fn from_rgba_f32(width, height, data) -> Self`
- 从已有 RGBA f32 数据构造纹理

#### `pub fn sample(&self, coord: Vec2f) -> Vec4f`
- **最近邻纹理采样**
- 步骤：
  1. 边界检查：宽高或数据为空返回 `(0,0,0,0)`
  2. UV 坐标 clamp 到 `[0, 1]`
  3. 转换到像素坐标：`px = floor(u * width)`，`py = floor(v * height)`
  4. 计算数据索引 `idx = (py * width + px) * 4`
  5. 边界检查后返回 RGBA 值
- 仅实现最近邻滤波，效率高但可能有锯齿

#### `pub fn sample_r8(&self, coord: Vec2f) -> Vec4f`
- **单通道纹理采样**
- 与 sample 相同但索引步长为 1（非 4）
- 返回 `(r, 0, 0, 1)`

---

## 5. 着色器内置自由函数

### 数学函数
| 函数 | 说明 |
|------|------|
| `mix_f32(a, b, t)` | 线性插值 `a + (b-a)*t` |
| `step(edge, x)` | 阶跃函数：`x < edge ? 0 : 1` |
| `smoothstep(edge0, edge1, x)` | 平滑阶跃：三次 Hermite 插值 `t*t*(3-2*t)` |
| `sign(x)` | 取符号：正→1、负→-1、零→0 |
| `sqrt(x)` | 平方根 |
| `inverse_sqrt(x)` | 反平方根 `1/sqrt(x)` |
| `inverse(m: Mat4f)` | 矩阵求逆，委托 `m.invert()` |
| `modf(x, y)` | 浮点取模：`x - y * floor(x/y)` |
| `atan2(y, x)` | 双参数反正切 |

### 向量函数
| 函数 | 说明 |
|------|------|
| `distance_2f(a, b)` | 二维欧几里得距离 |
| `length_2f(v)` | 二维向量模长 |
| `dot_2f(a, b)` | 二维向量点积 |
| `normalize_2f(v)` | 二维向量归一化（零向量返回零） |
| `dot_4f(a, b)` | 四维向量点积 |
| `length_4f(v)` | 四维向量模长 |

---

## 6. 颜色转换

### `pub fn hsv_to_rgb(hsv: Vec4f) -> Vec4f`
- HSV → RGB 颜色空间转换（GPU 风格实现）
- 算法：
  1. `c = v * s`（色度），`x = c * (1 - |(h*6) mod 2 - 1|)`，`m = v - c`
  2. 根据 h 在 6 个扇区的位置决定 (r,g,b) 的 (c,x,0) 分配
  3. 最终 RGB = (r+m, g+m, b+m)，A 保持不变
- 与 `math_f32.rs` 中 `Vec4f::from_hsva` 不同，此处是更直观的扇区分支实现

---

## 7. Sdf2d — 二维有符号距离场渲染器

### 类型定义
- **[pos]**: `Vec2f` — 当前像素位置（片元坐标）
- **[result]**: `Vec4f` — 累积渲染结果（预乘 alpha）
- **[last_pos / start_pos]**: `Vec2f` — 路径绘制中的上一个点和起点
- **[shape]**: `f32` — 当前形状的有符号距离（负值在内部，正值在外部）
- **[clip]**: `f32` — 裁剪距离
- **[has_clip]**: `bool` — 是否启用了裁剪
- **[old_shape]**: `f32` — 上一个操作后的 shape 值（用于布尔运算）
- **[blur]**: `f32` — 模糊半径（默认 0.00001）
- **[aa]**: `f32` — 抗锯齿宽度（默认 1.5 像素）
- **[scale_factor]**: `f32` — 缩放因子（默认 1.0）
- **[dist]**: `f32` — 通用距离字段

### `impl Sdf2d`

#### `pub fn viewport_f2(pos: Vec2f) -> Self`
- 构造函数，初始化 SDF 渲染器
- pos 设置当前片元位置
- shape 初始为极大值 `1e20`，clip 为极小值 `-1e20`
- aa 设为 1.5 像素用于抗锯齿计算

---

### 形状图元

#### `pub fn circle_f3(&mut self, x: f32, y: f32, r: f32)`
- **圆形 SDF**：计算点到圆心距离减去半径 `sqrt(dx*dx + dy*dy) - r`

#### `pub fn box_f4(&mut self, x: f32, y: f32, w: f32, h: f32, r: f32)`
- **圆角矩形 SDF**
- 先计算点到矩形中心距离，减去半宽半高
- 对超出部分使用 `hypot`（距离函数），再减去圆角半径 r

#### `pub fn rect_f4(&mut self, x: f32, y: f32, w: f32, h: f32)`
- **矩形 SDF**（无圆角）
- 点到矩形边界的距离：先计算到中心距离减去半宽半高，对负分量取 max(0)，再计算 hypot
- 内部区域（dx, dy 均 < 0）返回 max(dx, dy)

#### `pub fn hexagon_f3(&mut self, x: f32, y: f32, r: f32)`
- **正六边形 SDF**
- 利用六边形的对称性：取绝对值后用 `(dx*0.866 + dy*0.5).max(dy) - r`
- 0.866 = sqrt(3)/2，是六边形内角 120 度的几何常数

---

### 路径操作

#### `pub fn move_to(&mut self, x: f32, y: f32)`
- 移动到新位置，重置路径状态

#### `pub fn move_to_f2(&mut self, x: f32, y: f32)`
- move_to 的别名

#### `pub fn line_to(&mut self, x: f32, y: f32)`
- **线段 SDF**
- 步骤：
  1. 计算当前点到线段起点的向量 pa 和线段向量 ba
  2. 计算投影比率 h = clamp(dot(pa, ba) / dot(ba, ba), 0, 1)
  3. 计算垂距 d = length(pa - ba * h)
  4. shape 取当前 shape 和 d 的最小值（多条线段取最近者）

#### `pub fn close_path(&mut self)`
- 闭合路径：从 last_pos 到 start_pos 画一条线段

---

### 布尔操作

#### `pub fn union(&mut self)`
- 并集：`result = min(a, b)`（old_shape 和 shape 取最小距离）

#### `pub fn intersect(&mut self)`
- 交集：`result = max(a, b)`（取最大距离）

#### `pub fn subtract(&mut self)`
- 差集：`result = max(-b, a)`（将 b 取反后与 a 取并集）

---

### 填充与描边

#### `pub fn fill_keep(&mut self, color: Vec4f) -> Vec4f`
- **填充**（保持 shape 状态供后续操作）
- 步骤：
  1. 计算 alpha = clamp(-d/aa + 0.5, 0, 1)（距离越近 alpha 越高）
  2. 预乘 alpha 颜色：premul = (r*a, g*a, b*a, a)
  3. 使用混合公式累积到 result：`result = premul + result * (1 - premul.w)`
- "Keep" 版本不清除 shape，允许多次填充累积

#### `pub fn fill(&mut self, color: Vec4f) -> Vec4f`
- 填充后重置 shape/clip 状态

#### `pub fn stroke_keep(&mut self, color: Vec4f, width: f32) -> Vec4f`
- **描边**（保持状态）
- 取 shape 绝对值（描边在边缘两侧均匀扩展）
- 偏移 half-width 后计算 alpha，其余与 fill 相同

#### `pub fn stroke(&mut self, color: Vec4f, width: f32) -> Vec4f`
- 描边后重置状态

#### `pub fn glow_keep(&mut self, color: Vec4f, width: f32) -> Vec4f`
- **辉光/模糊描边**（保持状态）
- alpha = exp(-d^2 / (width^2 * 0.1))
- 使用高斯衰减而非线性衰减，产生柔和光晕效果

#### `pub fn glow(&mut self, color: Vec4f, width: f32) -> Vec4f`
- 辉光后重置状态
