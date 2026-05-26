# `math_f32.rs` 源码解读

**路径:** `libs/math/src/math_f32.rs`
**行数:** 2248
**核心职责:** Makepad 图形管线的 f32 精度数学核心库：向量、矩阵、四元数、位姿、平面、包围盒

---

## 类型别名和常量函数

### 向后兼容类型别名
- **`Vec2`**: `Vec2f` 的别名；**`Vec3`**: `Vec3f` 的别名；**`Vec4`**: `Vec4f` 的别名
- `vec2/vec3/vec4` — const fn 简写构造器

### 颜色预设常量
- **`VF00`**: `Vec4f{x:1,y:0,z:0,w:1}` — 红色通道掩码常量
- **`V0F0`**: `Vec4f{x:0,y:1,z:0,w:1}` — 绿色通道掩码常量
- **`V00F`**: `Vec4f{x:0,y:0,z:1,w:1}` — 蓝色通道掩码常量

### `PrettyPrintedF32`
- 包装 f32，实现 `Display` 使整数附近的浮点显示为 `N.0` 而非 `N`
- 判断逻辑：`self.0.abs().fract() < 0.00000001` 时附加 `.0` 后缀

---

## 类型定义

### `Vec2Index`
- 枚举 `{ X, Y }`，用于向量分量索引化访问

### `Pose`
- **[orientation]**: `Quat` — 旋转部分（单位四元数）
- **[position]**: `Vec3f` — 平移部分
- 表示三维空间中的一个完整位姿（刚体变换）

### `Vec2f`
- `{ x: f32, y: f32 }` — 二元向量

### `Vec3f`
- `{ x: f32, y: f32, z: f32 }` — 三元向量

### `Vec4f`
- `{ x: f32, y: f32, z: f32, w: f32 }` — 四元向量（常用于齐次坐标或 RGBA 颜色）

### `Plane`
- `{ a, b, c, d: f32 }` — 平面方程 `ax + by + cz + d = 0`

### `CameraFov`
- `{ angle_left, angle_right, angle_up, angle_down: f32 }` — 非对称视锥体定义

### `Quat`
- `{ x, y, z, w: f32 }` — 四元数，默认值为单位四元数 `(0,0,0,1)`

### `Mat4f`
- `{ v: [f32; 16] }` — 4x4 矩阵，公开存储为 16 元素一维数组，列主序

### `Mat3f`
- `{ c0, c1, c2: Vec3f }` — 3x3 矩阵，列主序（每个字段是一列）

### `Aabb`
- `{ min: Vec3f, max: Vec3f }` — 轴对齐包围盒

---

## impl 块详解

---

### `impl Pose`

#### `pub fn new(orientation: Quat, position: Vec3f) -> Self`
- 构造函数，用给定的旋转和平移创建位姿

#### `pub fn transform_vec3(&self, v: &Vec3f) -> Vec3f`
- 将向量 v 从局部坐标系变换到世界坐标系
- 先旋转（通过四元数旋转 v），再平移

#### `pub fn multiply(a: &Pose, b: &Pose) -> Self`
- 位姿合成：`result = a * b`（先应用 b 再应用 a）
- 旋转部分使用四元数乘法 `Quat::multiply(b.orientation, a.orientation)`
- 平移部分先对 b 的位置做 a 的变换再加上 a 的位置

#### `pub fn invert(&self) -> Self`
- 位姿求逆：若 Pose 表示 `T * R`，则逆为 `R^{-1} * T^{-1}`
- 旋转取逆（四元数共轭），平移旋转后取反

#### `pub fn to_mat4(&self) -> Mat4f`
- 将 Pose 转为 4x4 变换矩阵
- 左上 3x3 部分由四元数到旋转矩阵的公式填充
- 第 4 列为平移分量，第 4 行为 `[0,0,0,1]`

#### `pub fn from_lerp(a: Pose, b: Pose, f: f32) -> Self`
- 位姿线性插值：旋转用球面插值 `slerp`，平移用线性插值 `lerp`

#### `pub fn from_slerp_orientation(a: Pose, b: Pose, f: f32) -> Self`
- 仅旋转做 slerp，位置直接取 b 的位置。用于"朝向插值但位置固定"的场景

#### `pub fn is_finite(&self) -> bool`
- 检查 orientation 和 position 的所有分量是否有限

---

### `impl Vec2f`

#### `pub fn new() -> Vec2f`
- 默认构造零向量

#### `pub fn index(&self, index: Vec2Index) -> f32`
- 通过 `Vec2Index` 枚举索引访问 x 或 y

#### `pub fn set_index(&mut self, index: Vec2Index, v: f32)`
- 通过枚举设置 x 或 y

#### `pub fn from_index_pair(index: Vec2Index, a: f32, b: f32) -> Self`
- 根据索引构造：若为 X 则 `(a,b)`，若为 Y 则 `(b,a)`。用于分量交换操作

#### `pub fn into_vec2d(self) -> Vec2d`
- f32 向量转 f64 精度，分别做 as 转换

#### `pub fn all(x: f32) -> Vec2f`
- 构造所有分量相同的向量 `(x, x)`

#### `pub fn from_lerp(a: Vec2f, b: Vec2f, f: f32) -> Vec2f`
- 线性插值：`result = a * (1-f) + b * f`

#### `pub fn distance(&self, other: &Vec2f) -> f32`
- 欧几里得距离 `sqrt((dx)^2 + (dy)^2)`

#### `pub fn angle_in_radians(&self) -> f32`
- 向量相对于 x 轴的弧度角，使用 `atan2(y, x)`

#### `pub fn angle_in_degrees(&self) -> f32`
- 弧度角乘以 `360/(2*PI)` 转为度数

#### `pub fn length(&self) -> f32`
- 向量模长 `sqrt(x^2 + y^2)`

#### `pub fn lengthsquared(&self) -> f32`
- 向量模长平方 `x^2 + y^2`（避免开方，用于距离比较优化）

#### `pub fn normalize(&self) -> Vec2f`
- 归一化：各分量除以模长。零向量返回 `(0,0)`
- 先判零避免除以零

#### `pub fn normalize_to_x(&self) -> Vec2f`
- 以 x 分量为基准归一化，结果 x 恒为 1.0。若 x 为零则返回 `(1,0)`

#### `pub fn normalize_to_y(&self) -> Vec2f`
- 以 y 分量为基准归一化，结果 y 恒为 1.0。若 y 为零则返回 `(1,0)`

#### `pub fn to_vec3f(&self) -> Vec3f`
- 扩展为三维向量，z 设为 0.0

---

### `impl Vec3f`

#### `pub fn from_lerp(a: Vec3f, b: Vec3f, f: f32) -> Vec3f`
- 线性插值：每分量 `(b_i - a_i) * f + a_i`

#### `pub fn zero(&mut self)`
- 将所有分量置零

#### `pub const fn all(x: f32) -> Vec3f`
- 构造所有分量相同的向量 `(x, x, x)`

#### `pub const fn to_vec2(&self) -> Vec2f`
- 丢弃 z 分量转为二维向量

#### `pub const fn to_vec4(&self) -> Vec4f`
- 扩展为四维向量，w 设为 1.0（齐次坐标）

#### `pub fn scale(&self, f: f32) -> Vec3f`
- 标量乘法：所有分量乘以 f

#### `pub fn cross(a: Vec3f, b: Vec3f) -> Vec3f`
- 向量叉积。公式：`a × b = (a.y*b.z - a.z*b.y, a.z*b.x - a.x*b.z, a.x*b.y - a.y*b.x)`
- 结果垂直于 a 和 b 构成的平面，方向由右手定则确定

#### `pub fn dot(&self, other: Vec3f) -> f32`
- 向量点积：`x1*x2 + y1*y2 + z1*z2`，结果为标量

#### `pub fn normalize(&self) -> Vec3f`
- 归一化：长度平方 `sz > 0` 时用 `1/sqrt(sz)` 乘以各分量（使用一次开方、一次除法，比分别除更快）
- 零向量返回默认值 `(0,0,0)`

#### `pub fn length(&self) -> f32`
- 模长：`sqrt(x^2 + y^2 + z^2)`

#### `pub fn length_squared(&self) -> f32`
- 模长平方，避免开方的优化版本

#### `pub fn abs(&self) -> Vec3f`
- 每分量取绝对值

#### `pub fn min_elem(&self) -> f32`
- 三个分量中的最小值

#### `pub fn max_elem(&self) -> f32`
- 三个分量中的最大值

#### `pub fn min_componentwise(a: Vec3f, b: Vec3f) -> Vec3f`
- 逐分量取最小值

#### `pub fn max_componentwise(a: Vec3f, b: Vec3f) -> Vec3f`
- 逐分量取最大值

#### `pub fn is_finite(&self) -> bool`
- 三个分量均有限返回 true

---

### `impl Plane`

#### `pub fn from_point_normal(p: Vec3f, normal: Vec3f) -> Self`
- 由平面上一点和法线构造平面
- 首先归一化法线，然后 `d = -dot(p, n)`
- 平面方程系数 `(a,b,c,d)` 满足 `ax + by + cz + d = 0`

#### `pub fn from_points(p1: Vec3f, p2: Vec3f, p3: Vec3f) -> Self`
- 由三个不共线点构造平面
- 叉积计算法线 `(p2-p1) × (p3-p1)`，然后调用 `from_point_normal`

#### `pub fn intersect_line(&self, v1: Vec3f, v2: Vec3f) -> Vec3f`
- 计算平面与直线 `v1->v2` 的交点
- 参数 u = `(a*v1.x + b*v1.y + c*v1.z + d) / denom`，其中分母为 `a*dx + b*dy + c*dz`
- 若分母为零（直线平行于平面），返回 `(v1+v2)/2` 作为回退

---

### `impl Vec4f`

#### `pub const R / G / B` — 颜色常量
- R=(1,0,0,1), G=(0,1,0,1), B=(0,0,1,1)

#### `pub const fn all(v: f32) -> Self`
- 所有分量相同 `(v,v,v,v)`

#### `pub const fn to_vec3f(&self) -> Vec3f`
- 丢弃 w 分量取前三个分量

#### `pub fn dot(&self, other: Vec4f) -> f32`
- 四维点积：`x1*x2 + y1*y2 + z1*z2 + w1*w2`

#### `pub fn from_lerp(a: Vec4f, b: Vec4f, f: f32) -> Vec4f`
- 线性插值，每分量 `(1-f)*a_i + f*b_i`

#### `pub fn is_equal_enough(&self, other: &Vec4f, epsilon: f32) -> bool`
- 近似相等判断：每分量差值绝对值 < epsilon

#### `pub fn from_hsva(hsv: Vec4f) -> Vec4f`
- **HSV 到 RGB 颜色空间转换**。输入 `(h,s,v,a)` 输出 `(r,g,b,1.0)`
- 算法：利用分段三角函数，将色调 h 映射到色环上的六个扇区
- 核心公式：`v * mix(1, clamp(|fract(h + offset) * 6 - 3| - 1, 0, 1), s)`
- 对 R/G/B 分别偏移 0、2/3、1/3 相位

#### `pub fn to_hsva(&self) -> Vec4f`
- **RGB 到 HSV 反向转换**
- 通过逐步比较找出最大/最小通道
- 色调计算依赖色相扇区定位，饱和度 `d/q0`，明度 `q0`
- 使用 1e-10 避免零除

#### `pub fn from_u32(val: u32) -> Vec4f`
- 将 u32 编码的 RGBA 颜色（ARGB 格式）转为 `[0,1]` 范围的 Vec4f
- 使用移位和掩码：R=(val>>24)/255, G=(val>>16)/255, B=(val>>8)/255, A=(val&0xff)/255

#### `pub fn to_u32(&self) -> u32`
- Vec4f 颜色转 u32：每分量乘以 255 后转为 u8，按 ARGB 拼接

#### `pub const fn xy(&self) -> Vec2f`
- 取前两个分量

#### `pub const fn zw(&self) -> Vec2f`
- 取后两个分量

---

### `impl Quat`

#### `pub fn multiply(a: &Quat, b: &Quat) -> Self`
- **四元数乘法**。使用标准哈密尔顿积公式：
  - x = bw*ax + bx*aw + by*az - bz*ay
  - y = bw*ay - bx*az + by*aw + bz*ax
  - z = bw*az + bx*ay - by*ax + bz*aw
  - w = bw*aw - bx*ax - by*ay - bz*az
- 注意：四元数乘法不可交换，这里计算的是 `a * b`

#### `pub fn invert(&self) -> Self`
- **四元数逆**。对于单位四元数，逆等于共轭 `(-x, -y, -z, w)`

#### `pub fn rotate_vec3(&self, v: &Vec3f) -> Vec3f`
- **用四元数旋转向量**
- 构造纯四元数 `q=(v.x, v.y, v.z, 0)`
- 计算 `aq = q * self`，再计算 `ainv * aq`（其中 ainv 是 self 的逆）
- 结果 `aqainv` 的 xyz 即为旋转后的向量
- 原理：`R(v) = q⁻¹ * v * q`（其中 q 是单位四元数）

#### `pub fn dot(&self, other: Quat) -> f32`
- 四元数四维点积，用于度量四元数之间的夹角

#### `pub fn neg(&self) -> Quat`
- 四元数取负（q 和 -q 代表相同的旋转，但插值时符号很重要）

#### `pub fn get_angle_with(&self, other: Quat) -> f32`
- 计算两个四元数之间的角度差
- 公式：`acos(2*dot ^ 2 - 1) * TODEG`
- 利用四元数点积的平方间接计算角度

#### `pub fn from_slerp(n: Quat, mut m: Quat, t: f32) -> Quat`
- **球面线性插值（Slerp）**
- 步骤：
  1. 计算点积 `cosom = n.dot(m)`
  2. 若 cosom < 0，取负 m 以保证最短路径插值
  3. 若接近 1（角度很小），退化为线性插值（Lerp）避免零除
  4. 否则计算 omega = acos(cosom)，sinom = sin(omega)，使用标准 slerp 系数：`sin((1-t)*omega)/sinom` 和 `sin(t*omega)/sinom`
  5. 对结果调用 `normalized()` 确保单位长度

#### `pub fn length(self) -> f32`
- 四元数模长 `sqrt(dot(self))`

#### `pub fn is_finite(&self) -> bool`
- 四个分量均有限

#### `pub fn normalized(&mut self) -> Quat`
- 归一化为单位四元数。各分量除以模长

#### `pub fn from_axis_angle(axis: Vec3f, angle: f32) -> Self`
- **轴角转四元数**
- 半角公式：`q = (axis * sin(angle/2), cos(angle/2))`
- 要求轴向量为单位向量

#### `pub fn integrate(&self, angular_velocity: Vec3f, dt: f32) -> Quat`
- **一阶四元数积分**，用于物理模拟中刚体角速度积分
- 构造纯四元数 omega = (wx, wy, wz, 0)
- 计算 omega_q = self * omega（注意乘法顺序）
- 更新：`result = self + 0.5 * dt * omega_q`
- 归一化保持单位长度
- 不需要三角函数调用，效率高

#### `pub fn look_rotation(forward: Vec3f, up: Vec3f) -> Self`
- **从视线方向构造旋转四元数**（类似 LookAt 的纯旋转版本）
- 步骤：
  1. 归一化 forward 和 up
  2. 计算右向量 v0 = cross(up, forward)，上向量 v1 = cross(v2, v0)
  3. 构造旋转矩阵 `[v0, v1, v2]^T` 的 3x3 部分
  4. 根据旋转矩阵到四元数的转换公式分四种情况：
     - 当迹 num = v0.x + v1.y + v2.z > 0 时使用标准公式
     - 否则根据主对角线最大元素分三种情况处理，避免数值不稳定
  5. 每种情况计算对应的四元数分量

---

### `impl Mat4f`

#### `pub const fn identity() -> Mat4f`
- 单位矩阵，对角线为 1，其余为 0

#### `pub fn transpose(&self) -> Mat4f`
- 矩阵转置：v[i][j] ↔ v[j][i]
- 列主序下，转置即交换行列索引

#### `pub fn txyz_s_ry_rx_txyz(t1, s, ry, rx, t2) -> Mat4f`
- **复合变换矩阵**：先平移 t1，再均匀缩放 s，再绕 Y 轴旋转 ry 度，再绕 X 轴旋转 rx 度，再平移 t2
- 手动展开 `Rx * Ry` 的旋转矩阵组合，避免计算 Z 旋转（为零）
- 平移分量包含 t1 经过旋转缩放后加上 t2 的结果

#### `pub fn perspective(fov_y: f32, aspect: f32, near: f32, far: f32) -> Mat4f`
- **透视投影矩阵**（标准 GL 风格）
- f = 1/tan(fov_y/2)，nf = 1/(near - far)
- 构建列主序投影矩阵：x 缩放 `f/aspect`，y 缩放 `f`，z 部分映射到 [0, 1] 或 [-1, 1]

#### `pub fn from_camera_fov(fov: &CameraFov, near: f32, far: f32) -> Mat4f`
- **非对称视锥体投影矩阵**，用于 VR/AR 或 Off-axis 投影
- 从 CameraFov 的四个角度计算 tan 值
- tan_height/tan_width 确定投影缩放
- 平移分量将视锥中心对齐到投影中心
- 分 infinite far (far <= near) 和有限 far 两种情况

#### `pub const fn translation(v: Vec3f) -> Mat4f`
- 纯平移矩阵，左上 3x3 为单位阵，第 4 列为平移分量

#### `pub const fn nonuniform_scaled_translation(s: Vec3f, t: Vec3f) -> Mat4f`
- 非均匀缩放 + 平移复合矩阵

#### `pub const fn scaled_translation(s: f32, t: Vec3f) -> Mat4f`
- 均匀缩放 + 平移复合矩阵

#### `pub const fn scale(s: f32) -> Mat4f`
- 纯均匀缩放矩阵，对角线为 s

#### `pub fn rotation(r: Vec3f) -> Mat4f`
- **欧拉角旋转矩阵**（ZYX 顺序）
- 输入弧度值，按 Z→X→Y 顺序组合旋转
- 手动展开三个基本旋转矩阵的乘积公式

#### `pub fn ortho(left, right, top, bottom, near, far, scalex, scaley) -> Mat4f`
- **正交投影矩阵**
- 将世界坐标映射到裁剪空间
- lr = 1/(left-right)，bt = 1/(bottom-top)，nf = 1/(near-far)
- 包含 scalex/scaley 参数用于 DPI 缩放适配

#### `pub fn transform_vec4(&self, v: Vec4f) -> Vec4f`
- **矩阵乘以列向量**：4x4 矩阵与 4 维向量的标准乘法
- 每分量等于矩阵对应行与向量的点积

#### `pub fn look_at(eye: Vec3f, center: Vec3f, up: Vec3f) -> Mat4f`
- **视图矩阵**（LookAt）
- 步骤：
  1. forward = normalize(center - eye)
  2. side = normalize(cross(forward, up))
  3. up = cross(side, forward)
  4. 构建以 side/up/-forward 为基的旋转矩阵，平移为 `-side·eye`、`-up·eye`、`forward·eye`

#### `pub fn mul(a: &Mat4f, b: &Mat4f) -> Mat4f`
- **矩阵乘法**。计算 `a * b`（标准线性代数顺序，不是反转的）
- 内部交换 a/b 引用以处理列主序布局：`(b * a)` 因为列主序下数组存储和数学意义之间有转置关系
- 使用辅助函数 `d(i, x, y) = i[x + 4*y]` 进行列索引提取
- 展开全部 16 个元素的乘加计算

#### `pub fn invert(&self) -> Mat4f`
- **矩阵求逆**，使用代数余子式（cofactor）方法
- 步骤：
  1. 计算 12 个 2x2 子行列式 (b00-b11)
  2. 由这些子行列式计算 4x4 行列式 det
  3. 若 det=0 返回单位矩阵
  4. 计算伴随矩阵的各个元素，乘以 1/det
- 复杂度为 16 次乘加 + 1 次除法，无循环展开

---

### `impl Mat3f`

#### `pub const fn identity() -> Self`
- 3x3 单位矩阵

#### `pub const fn zero() -> Self`
- 3x3 零矩阵

#### `pub const fn from_diagonal(d: Vec3f) -> Self`
- 对角矩阵，用于刚体惯性张量等场景

#### `pub fn from_quat(q: Quat) -> Self`
- 单位四元数转 3x3 旋转矩阵
- 标准公式：对四元数分量计算 xx、xy、xz、yy、yz、zz、wx、wy、wz
- 填充矩阵的三个列向量

#### `pub fn transpose(&self) -> Self`
- 3x3 转置：列向量互换

#### `pub fn mul_vec3(&self, v: Vec3f) -> Vec3f`
- 3x3 矩阵乘以三维向量

#### `pub fn mul_mat3(&self, rhs: &Mat3f) -> Mat3f`
- 3x3 矩阵乘法：对 rhs 的每一列调用 mul_vec3

#### `pub fn scale(&self, s: f32) -> Self`
- 矩阵的每个元素乘以标量。等价于 `self * diag(s)`

---

### `impl Aabb`

#### `pub fn overlaps(&self, other: &Aabb) -> bool`
- AABB 相交测试。六面比较：各轴上的 `min <= other.max` 且 `max >= other.min`
- 三轴均通过则返回 true

#### `pub fn from_cuboid(half_extents: Vec3f, pose: &Pose) -> Self`
- 从半长和位姿计算有向物体的 AABB
- 使用"绝对值旋转矩阵"技巧：将半长乘以旋转矩阵各列的绝对值，得到各轴上的投影范围
- AABB 中心在位姿位置，范围由投影长度确定

---

## 运算符重载

### Vec2f 运算符
- **加减乘除 (Vec2f vs Vec2f, f32 vs Vec2f, Vec2f vs f32)**: 逐分量运算
- **Assign 版本**: `+=`, `-=`, `*=`, `/=` — 就地修改
- **Neg**: 逐分量取反

### Vec3f 运算符
- 与 Vec2f 相同的全量运算符重载模式，逐分量运算

### Vec4f 运算符
- 与 Vec2f 相同的全量运算符重载模式，逐分量运算
- 注意：`f32 + Vec4f` 的 w 分量有 bug：写成 `rhs.z` 而非 `rhs.w`，但其他同类重载中 w 处理正确

---

## 单元测试

### `mat4_mul_order`
- 验证 `Mat4f::mul(a, b)` 是标准 `a * b` 顺序而非反转
- 构造缩放矩阵和位移矩阵，通过变换向量验证结果
- 期望：`Scale(2) * Translate(5,7)` 作用于点 (1,1) 得 (12,16)
