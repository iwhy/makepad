# `complex.rs` 源码解读

**路径:** `libs/math/src/complex.rs`
**行数:** 127
**核心职责:** 提供 f32/f64 复数类型及基本 FFT 变换实现

---

## 类型定义

### `ComplexF32`
- **[re]**: `f32` — 实部
- **[im]**: `f32` — 虚部

### `ComplexF64`
- **[re]**: `f64` — 实部
- **[im]**: `f64` — 虚部

## impl 块

### `impl ComplexF32`
- **`pub fn magnitude(self) -> f32`**: 计算复数模长。利用勾股定理 `sqrt(re*re + im*im)`，这是复数模长的标准定义。

### `impl From<ComplexF32> for ComplexF64`
- **`fn from(v: ComplexF32) -> Self`**: f32 复数向 f64 复数的高精度提升转换，两分量分别做 as 类型转换。

### `impl From<ComplexF64> for ComplexF32`
- **`fn from(v: ComplexF64) -> Self`**: f64 复数向 f32 复数的降精度转换，两分量分别做 as 截断。

### `impl Mul<ComplexF64> for ComplexF64`
- **`fn mul(self, rhs) -> ComplexF64`**: 复数乘法。公式 `(a+bi)(c+di) = (ac-bd) + (ad+bc)i`，展开后实部为 `self.re*rhs.re - self.im*rhs.im`，虚部为 `self.re*rhs.im + self.im*rhs.re`。

### `impl Add<ComplexF64> for ComplexF64`
- **`fn add(self, rhs) -> ComplexF64`**: 复数加法。实部和虚部分别相加。

### `impl Sub<ComplexF64> for ComplexF64`
- **`fn sub(self, rhs) -> ComplexF64`**: 复数减法。实部和虚部分别相减。

## 函数

### `pub const fn cf64(re: f64, im: f64) -> ComplexF64`
- const fn 构造函数，编译期可求值。创建 ComplexF64 实例。

### `pub const fn cf32(re: f32, im: f32) -> ComplexF32`
- const fn 构造函数，编译期可求值。创建 ComplexF32 实例。

---

## FFT 实现

该模块包含一个基于 Cooley-Tukey 蝶形算法的 FFT 实现，从 `https://github.com/rshuston/FFT-C/` 移植。

### `fn fft_f32_recursive_pow2_inner(data, scratch, n, theta_pi, stride)`
- **递归蝶形变换核心**（非 pub）。
- 算法步骤：
  1. 递归基条件：当 `stride < n` 时继续分解
  2. 蝶形步长翻倍 `stride2 = stride * 2`
  3. 将 data 分为奇偶两部分递归调用（交替使用 data 和 scratch 作为输入输出，避免就地覆盖）
  4. 计算旋转因子 `wn = e^(i*theta)`，其中 `theta = stride2 * theta_pi / n`
  5. 对 `k in (0..n).step_by(stride2)` 执行蝶形操作：`u = scratch[k]`，`t = wnk * scratch[k+stride]`，然后 `data[k>>1] = u+t`，`data[(k+n)>>1] = u-t`
  6. 每完成一个蝶形后更新旋转因子 `wnk *= wn`
- 交替使用 data 和 scratch 是原地 FFT 的经典技巧，避免了额外内存分配。

### `fn fft_f32_recursive_pow2(data, scratch, theta_pi)`
- FFT 入口调度函数，非 pub。
- 断言 data 和 scratch 等长
- 检查长度是否为 2 的幂（`n != 0 && (!(n & (n - 1))) != 0`）
- 将 data 复制到 scratch 后调用 `fft_f32_recursive_pow2_inner`
- `theta_pi` 控制变换方向：`-PI` 为正向，`+PI` 为逆向

### `pub fn fft_f32_recursive_pow2_forward(data, scratch)`
- **正向 FFT**：以 `theta_pi = -PI` 调用 `fft_f32_recursive_pow2`

### `pub fn fft_f32_recursive_pow2_inverse(data, scratch)`
- **逆向 FFT**：以 `theta_pi = +PI` 调用 `fft_f32_recursive_pow2`
