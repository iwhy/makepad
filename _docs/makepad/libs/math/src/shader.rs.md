# `shader.rs` 源码解读

**路径:** `libs/math/src/shader.rs`
**行数:** 316
**核心职责:** 定义 ShaderMath trait 统一标准数学函数接口，使 CPU 端数学运算与 GPU 着色器语法兼容

---

## 设计目标

- 提供一组与 GLSL 内置函数同名的 trait 方法，让 CPU 端代码可以直接使用与着色器相同的数学函数名
- 为 `f32` 和 `f64` 实现该 trait，实现 CPU 端数学函数的一层薄封装

---

## Trait 定义

### `ShaderMath`

#### 三角函数
| 方法 | 等价 GLSL | 说明 |
|------|-----------|------|
| `abs(self) -> Self` | `abs` | 绝对值 |
| `sin(self) -> Self` | `sin` | 正弦 |
| `cos(self) -> Self` | `cos` | 余弦 |
| `tan(self) -> Self` | `tan` | 正切 |
| `asin(self) -> Self` | `asin` | 反正弦 |
| `acos(self) -> Self` | `acos` | 反余弦 |
| `atan(self) -> Self` | `atan` | 反正切 |

#### 双曲函数
| 方法 | 等价 GLSL | 说明 |
|------|-----------|------|
| `sinh / cosh / tanh` | `sinh / cosh / tanh` | 双曲正弦/余弦/正切 |
| `asinh / acosh / atanh` | `asinh / acosh / atanh` | 反双曲函数 |

#### 实用函数
| 方法 | 等价 GLSL | 说明 |
|------|-----------|------|
| `fract(self) -> Self` | `fract` | 小数部分 |
| `ceil(self) -> Self` | `ceil` | 向上取整 |
| `floor(self) -> Self` | `floor` | 向下取整 |
| `min(self, v) -> Self` | `min` | 最小值 |
| `max(self, v) -> Self` | `max` | 最大值 |
| `clamp(self, low, high) -> Self` | `clamp` | 约束到 [low, high] |
| `exp(self) -> Self` | `exp` | 自然指数 e^x |
| `exp2(self) -> Self` | `exp2` | 2^x |
| `ln(self) -> Self` | `log` | 自然对数 |
| `log2(self) -> Self` | `log2` | 以 2 为底对数 |
| `log10(self) -> Self` | `log10` | 以 10 为底对数 |
| `powf(self, v) -> Self` | `pow` | 幂函数 x^y |
| `powi(self, p: i32) -> Self` | — | 整数指数幂 |

---

## 实现

### `impl ShaderMath for f32`
- 每个方法直接委托到 f32 的内置方法

### `impl ShaderMath for f64`
- 每个方法直接委托到 f64 的内置方法

---

## 自由函数

- 所有 trait 方法对应同名自由函数（如 `pub fn sin<T: ShaderMath>(v: T) -> T`）
- 提供类似 GLSL 全局函数的调用方式
- 额外提供了 `pow` 作为 `powf` 的别名
- 末尾注释列出了 GLSL 中常用但未实现的函数（cross、distance、dot、inverse、length、normalize），这些在其他模块中实现
