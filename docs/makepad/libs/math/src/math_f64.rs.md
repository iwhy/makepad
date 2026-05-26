# `math_f64.rs` 源码解读

**路径:** `libs/math/src/math_f64.rs`
**行数:** 576
**核心职责:** 双精度 (f64) 向量类型 Vec2d、矩形 Rect 及其运算

---

## 类型定义

### `PrettyPrintedF64`
- 包装 f64，实现 `Display` 使整数附近的浮点数显示为 `N.0` 而非 `N`

### `Rect`
- **[pos]**: `Vec2d` — 矩形左上角位置
- **[size]**: `Vec2d` — 矩形宽高
- 核心 UI 布局类型

### `Vec4d` / `DVec4`
- `{ x, y, z, w: f64 }` — 四维双精度向量

### `Vec3d` / `DVec3`
- `{ x, y, z: f64 }` — 三维双精度向量

### `Vec2d` / `DVec2`
- `{ x, y: f64 }` — 二维双精度向量

---

## impl 块详解

### `impl Rect`

#### `pub fn translate(self, pos: Vec2d) -> Rect`
- 平移矩形：位置加上偏移量，尺寸不变

#### `pub fn contains(&self, pos: Vec2d) -> bool`
- 点包含测试：pos 的 x/y 均在 `[pos, pos+size]` 范围内

#### `pub fn center(&self) -> Vec2d`
- 计算矩形中心：`pos + size * 0.5`

#### `pub fn scale_and_shift(&self, center: Vec2d, scale: f64, shift: Vec2d) -> Rect`
- 先围绕 center 缩放，再平移 shift
- 位置变换公式：`(pos - center) * scale + center + shift`

#### `pub fn is_inside_of(&self, r: Rect) -> bool`
- 判断 self 是否完全在 r 内部：四个边界全部包含

#### `pub fn intersects(&self, r: Rect) -> bool`
- AABB 相交测试：四边分别比较，任一方向无重叠则不相交

#### `pub fn add_margin(self, size: Vec2d) -> Rect`
- 向外扩展边距：位置减 size，尺寸加 2*size

#### `pub fn contain(&self, other: Rect) -> Rect`
- 将 other 约束在 self 边界内：若 other 超出 self 边界则平移，尺寸不变

#### `pub fn hull(&self, other: Rect) -> Rect`
- 计算两个矩形的并集包围盒：取最小 pos 和最大 far side

#### `pub fn clip(&self, clip: (Vec2d, Vec2d)) -> Rect`
- 用两个角点 (min, max) 裁剪矩形。取各边在 clip 范围内的交集

#### `pub fn from_lerp(a: Rect, b: Rect, f: f64) -> Rect`
- 矩形线性插值：pos 和 size 分别做 `(b-a)*f + a`

#### `pub fn dpi_snap(&self, f: f64) -> Rect`
- DPI 对齐/吸附：位置和尺寸按 `floor(val / f) * f` 取整，确保像素对齐

#### `pub fn grow(&mut self, amt: f64)`
- 就地扩展矩形：四边各增加 amt

#### `pub fn clip_y_between(&mut self, y1: f64, y2: f64)`
- 沿 Y 轴方向裁剪到 `[y1, y2]` 范围内

#### `pub fn clip_x_between(&mut self, x1: f64, x2: f64)`
- 沿 X 轴方向裁剪到 `[x1, x2]` 范围内

#### `pub fn is_nan(&self) -> bool`
- 检查 pos 或 size 是否包含 NaN

---

### `impl Vec2d`

#### `pub const fn new() -> Vec2d`
- 构造零向量 `(0, 0)`

#### `pub fn zero(&mut self)`
- 就地置零

#### `pub fn dpi_snap(&self, f: f64) -> Vec2d`
- DPI 对齐：`(val * f).round() / f`，比 Rect 的版本多了一步四舍五入

#### `pub const fn all(x: f64) -> Vec2d`
- 构造各分量相同的向量

#### `pub const fn index(&self, index: Vec2Index) -> f64`
- 通过 Vec2Index 枚举访问分量

#### `pub fn set_index(&mut self, index: Vec2Index, v: f64)`
- 通过枚举设置分量

#### `pub const fn from_index_pair(index: Vec2Index, a: f64, b: f64) -> Self`
- 根据索引交换顺序构造

#### `pub const fn into_vec2(self) -> Vec2f`
- f64 降精度为 f32

#### `pub fn from_lerp(a: Vec2d, b: Vec2d, f: f64) -> Vec2d`
- 线性插值

#### `pub fn floor(self) -> Vec2d`
- 每分量向下取整

#### `pub fn ceil(self) -> Vec2d`
- 每分量向上取整

#### `pub fn distance(&self, other: &Vec2d) -> f64`
- 欧几里得距离

#### `pub fn angle_in_radians(&self) -> f64`
- 相对于 x 轴的弧度角

#### `pub fn swapxy(&self) -> Vec2d`
- 交换 x 和 y。返回 `(y, x)`

#### `pub fn angle_in_degrees(&self) -> f64`
- 相对于 x 轴的角度（度）

#### `pub fn length(&self) -> f64`
- 模长

#### `pub fn normalize(&self) -> Vec2d`
- 归一化，零向量返回 `(0,0)`

#### `pub fn clockwise_tangent(&self) -> Vec2d`
- 顺时针切向量 `(-y, x)`，即旋转 90 度

#### `pub fn counterclockwise_tangent(&self) -> Vec2d`
- 逆时针切向量 `(y, -x)`

#### `pub fn normalize_to_x(&self) -> Vec2d`
- 以 x 为基准归一化

#### `pub fn normalize_to_y(&self) -> Vec2d`
- 以 y 为基准归一化

#### `pub fn lengthsquared(&self) -> f64`
- 模长平方

#### `pub fn is_nan(&self) -> bool`
- 检查是否存在 NaN

---

## 运算符重载

### Vec2d 全量运算符
- **加减乘除 (Vec2d vs Vec2d, f64 vs Vec2d, Vec2d vs f64)**: 逐分量运算
- **Assign 版本**: `+=`, `-=`, `*=`, `/=`
- **Neg**: 逐分量取反

## Type Conversion

### `From<Vec2f> for Vec2d`
- f32 → f64 提升转换

### `From<Vec2d> for Vec2f`
- f64 → f32 降级转换

### `From<(Vec2d, Vec2d)> for Rect`
- 两个角点 `(top-left, bottom-right)` 转为 Rect（pos + size 格式）

## 常量函数

### `pub const fn dvec2(x: f64, y: f64) -> Vec2d`
- Vec2d const 构造器

### `pub const fn rect(x: f64, y: f64, w: f64, h: f64) -> Rect`
- Rect const 构造器
