# `mod_math.rs` — Makepad Script 数学库模块

## 文件概述

`mod_math.rs` 是 Makepad Script 虚拟机中 `math` 模块的入口。它本身很短——只有 10 行——因为所有数学函数的具体实现在 `shader_builtins.rs` 中。`mod_math.rs` 负责创建 `math` 模块对象并委托 `define_shader_builtins` 填充函数。

---

## `define_math_module(heap, native)`

```rust
pub fn define_math_module(heap: &mut ScriptHeap, native: &mut ScriptNative) {
    let math = heap.new_module(id!(math));
    define_shader_builtins(heap, math, native);
}
```

**实现逻辑**:
1. 在堆上创建一个名为 `math` 的模块对象（通过 `id!(math)` 的 LiveId）
2. 调用 `define_shader_builtins(heap, math, native)`，传入刚创建的 `math` 对象作为目标模块
3. 所有数学函数（三角函数、指数、向量运算等）被注册到 `math` 对象上

---

## 注册的数学常量 (`shader_builtins.rs:36-49`)

| 名称 | 值 |
|------|-----|
| `PI` | `3.141592653589793` |
| `E` | `2.718281828459045` |
| `LN2` | `0.6931471805599453` |
| `LN10` | `2.302585092994046` |
| `LOG2E` | `1.4426950408889634` |
| `LOG10E` | `0.4342944819032518` |
| `SQRT1_2` | `0.70710678118654757` |
| `TORAD` | `0.017453292519943295`（π/180，度转弧度乘数） |
| `GOLDEN` | `1.618033988749895`（黄金比例 φ） |

这些常量通过循环 `heap.set_value_def(math, id.into(), val.into())` 注册。

---

## 注册的数学函数

所有函数通过 `native.add_method()` 注册到 `math` 模块。函数参数使用 `NumericValue` 类型系统，支持 `f64`、`Vec2f`、`Vec3f`、`Vec4f`、`Color` 等类型。

### 单参数函数（元素级操作）

通过统一的 `NumericValue::from_script_value_vm` → `.map_f32(|v| ...)` → `.to_script_value_vm` 流程实现。

| 函数 | 数学含义 | 实现 |
|------|----------|------|
| `abs(x)` | 绝对值 | `v.abs()` |
| `acos(x)` | 反余弦 | `v.acos()` |
| `acosh(x)` | 反双曲余弦 | `v.acosh()` |
| `asin(x)` | 反正弦 | `v.asin()` |
| `asinh(x)` | 反双曲正弦 | `v.asinh()` |
| `atan(x)` | 反正切（单参数） | `v.atan()` |
| `atanh(x)` | 反双曲正切 | `v.atanh()` |
| `ceil(x)` | 上取整 | `v.ceil()` |
| `cos(x)` | 余弦 | `v.cos()` |
| `cosh(x)` | 双曲余弦 | `v.cosh()` |
| `degrees(x)` | 弧度转角度 | `v.to_degrees()` |
| `exp(x)` | 指数函数 eˣ | `v.exp()` |
| `exp2(x)` | 2 的幂 2ˣ | `v.exp2()` |
| `floor(x)` | 下取整 | `v.floor()` |
| `fract(x)` | 小数部分 | `v.fract()` |
| `inverseSqrt(x)` | 平方根的倒数 1/√x | `v.sqrt().recip()` |
| `inverse(x)` | 矩阵求逆 | 仅对 `Mat4` 类型有效，调用 `Mat4f::invert()` |
| `length(x)` | 向量长度 | `nv.length()` 返回标量 f64 |
| `log(x)` | 自然对数 ln(x) | `v.ln()` |
| `log2(x)` | 以 2 为底的对数 | `v.log2()` |
| `radians(x)` | 角度转弧度 | `v.to_radians()` |
| `round(x)` | 四舍五入 | `v.round()` |
| `sign(x)` | 符号函数 | `v > 0 → 1.0, v < 0 → -1.0, else → 0.0` |
| `sin(x)` | 正弦 | `v.sin()` |
| `sinh(x)` | 双曲正弦 | `v.sinh()` |
| `sqrt(x)` | 平方根 | `v.sqrt()` |
| `tan(x)` | 正切 | `v.tan()` |
| `tanh(x)` | 双曲正切 | `v.tanh()` |
| `trunc(x)` | 截断取整 | `v.trunc()` |
| `normalize(x)` | 归一化向量 | `nv.normalize()` 返回同方向单位向量 |

### 双参数函数

通过 `.zip_f32()` 实现两个 `NumericValue` 的元素级操作：

| 函数 | 参数 | 实现 |
|------|------|------|
| `atan2(y, x)` | 2 个 | `y.atan2(x)` — 带象限的反正切 |
| `distance(x, y)` | 2 个 | 计算 `length(x - y)`，即两点间欧几里得距离，返回标量 |
| `dot(x, y)` | 2 个 | 计算 `x · y` 点积，各分量乘积之和，返回标量 |
| `cross(x, y)` | 2 个 | 三维叉积，仅对 `vec3` 类型有效 |
| `max(x, y)` | 2 个 | `a.max(b)` 元素级最大值 |
| `min(x, y)` | 2 个 | `a.min(b)` 元素级最小值 |
| `pow(x, y)` | 2 个 | `a.powf(b)` — x 的 y 次幂 |
| `modf(x, y)` | 2 个 | `a % b` — 浮点数取模 |
| `step(edge, x)` | 2 个 | `x < edge → 0, else → 1`，标量/向量混合支持 |

**`step()` 特殊处理**: 如果 `edge` 是标量但 `x` 是向量，调用 `NumericValue::step_scalar(edge_f, x_nv)` 实现标量到向量的广播。

### 三参数函数

| 函数 | 参数 | 实现 |
|------|------|------|
| `clamp(x, min, max)` | 3 个 | `x.max(min).min(max)`，支持标量/向量混合 |
| `mix(x, y, a)` | 3 个 | `x * (1-a) + y * a` 线性插值 |
| `smoothstep(e0, e1, x)` | 3 个 | Hermite 平滑插值 `t²(3-2t)`，`t = clamp((x-e0)/(e1-e0), 0, 1)` |
| `fma(a, b, c)` | 3 个 | `a * b + c` 乘加融合 |

**`mix()` 实现细节**: 
- 当 alpha 是标量时，使用 `x_nv.mix_scalar(y_nv, a_f)` 
- 当 alpha 是向量时，针对 `Vec2`/`Vec3`/`Vec4`/`Color` 做分量级混合
- 有完善的 fallback 机制处理类型不匹配

**`smoothstep()` 实现细节**:
- 支持标量 edge + 向量 x 的广播模式
- 支持向量 edge + 向量 x 的分量级计算
- 对 `Vec2`/`Vec3`/`Vec4`/`Color` 分别实现分量级 smoothstep
- Hermite 曲线公式：`t = clamp((x - e0) / (e1 - e0), 0, 1)`，`result = t * t * (3 - 2 * t)`

**`clamp()` 实现细节**:
- 当 min/max 为标量时使用 `clamp_scalar`
- 当 min/max 为向量时使用 `zip_f32` 链式调用 `x.max(m)` 再 `.min(m)`

### 着色器专用函数

| 函数 | 参数 | 运行时行为 | 说明 |
|------|------|-----------|------|
| `dFdx(x)` | 1 | 返回零值（同类型） | 屏幕空间 x 偏导数，仅在着色器中有效 |
| `dFdy(x)` | 1 | 返回零值（同类型） | 屏幕空间 y 偏导数，仅在着色器中有效 |
| `discard()` | 0 | `NIL` | 片段丢弃，仅在片段着色器中有效 |

### 位转换函数

| 函数 | 用途 | 实现 |
|------|------|------|
| `asuint(x)` | 将值重新解释为 uint | 尝试 `as_u32()` → `as_i32() as u32` → `f32::to_bits()` → `f16::to_bits()` |
| `asint(x)` | 将值重新解释为 int | 尝试 `as_i32()` → `as_u32() as i32` → `f32::to_bits() as i32` → `f16::to_bits() as i32` |
| `asfloat(x)` | 将值重新解释为 float | 尝试 `as_f32()` → `f32::from_bits(u32)` → `f32::from_bits(i32 as u32)` |

---

## `type_table_builtin(name, args, builtins, trap)` — 着色器编译期类型推导

除了运行时执行，`shader_builtins.rs` 还提供了 `type_table_builtin()` 函数用于 Shader 编译器的**类型检查**。它为每个内置函数定义了输入类型和返回值类型的关系：

**返回值规则**:
- 对大多单参数函数：输入什么浮点/向量类型 → 输出相同类型
- `length` / `distance` / `dot`：输入向量 → 输出标量（`f32` 或 `f16`）
- `normalize`：输入向量 → 输出同类型向量
- `inverse`：输入 `mat4` → 输出 `mat4`
- `cross`：输入两个 `vec3` → 输出 `vec3`
- `clamp` / `max` / `min`：支持浮点或整数类型
- `step`：支持 `(scalar, vec)` 广播
- `mix`：支持 `(vec, vec, scalar)` 标量 alpha
- `smoothstep` / `fma`：三个参数必须类型一致
- `discard`：`void`（无返回值）
- `depth_clip`：特定签名 `(vec4f, vec4f, float) → vec4f`

类型不匹配时通过 `script_err_*!` 宏报告详细错误，包括实际参数类型的格式化描述。

---

## 设计要点总结

1. **双模式设计**：运行时用 `NumericValue` 动态分发，编译期用 `type_table_builtin` 做静态类型检查
2. **向量/标量广播**：`step` / `clamp` / `mix` / `smoothstep` 均支持 scalar+vector 混合操作
3. **着色器兼容**：`dFdx`/`dFdy`/`discard` 在脚本运行时返回空值/零值，着色器编译时才真正生效
4. **位精确转换**：`asuint`/`asint`/`asfloat` 保持 IEEE 754 位模式，支持 f16↔u32 等跨类型重解释
5. **Mat4 求逆**：`inverse` 仅对 4×4 矩阵有效，非矩阵类型直接透传
6. **全面错误报告**：类型检查函数对参数数量/类型不匹配生成精确的错误消息
