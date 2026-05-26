# shader_builtins.rs — 着色器内置数学函数的运行时实现与编译期类型检查

**文件路径**: `platform/script/src/shader_builtins.rs`  
**行数**: 1553 行  
**作用**: 定义 Makepad 着色器 DSL 中所有内置数学函数的两个系统：运行时实现（通过 Native 方法提供给脚本 VM 执行）和编译期类型检查（通过 `type_table_builtin` 函数为着色器编译器提供类型推导和错误检查）。

---

## 辅助特质

### `NumericValueVmExt`（第 15–28 行）
为 `NumericValue` 添加 VM 上下文转换方法：
- `from_script_value_vm`: 从 `ScriptValue` 到 `NumericValue`，使用 VM 堆的 Pod 编译器
- `to_script_value_vm`: 从 `NumericValue` 到 `ScriptValue`，需要可变 VM 堆和代码引用

---

## `define_shader_builtins`（第 30–1069 行）

这是文件的主函数，将所有内置数学函数注册到脚本堆的 `math` 模块对象上。函数通过 `native.add_method` 注册，使脚本 VM 能在运行时调用这些函数。

### 数学常量（第 36–49 行）
注册九个标准数学常量到 `math` 对象：
- **PI**（3.14159）、**E**（2.71828）、**LN2**（0.69315）、**LN10**（2.30259）
- **LOG2E**（1.44270）、**LOG10E**（0.43429）、**SQRT1_2**（0.70711）
- **TORAD**（0.01745）、**GOLDEN**（1.61803）

这些常量在着色器编译时会被 `ShaderFnCompiler::shader_math_const_value` 内联为字面量。

### 一元浮点函数（第 51–517 行）
使用 `NumericValue::map_f32` 模式实现的 30+ 个函数，支持 f32、Vec2f、Vec3f、Vec4f 输入类型：

| 函数 | 操作 | 说明 |
|------|------|------|
| `abs` | `v.abs()` | 绝对值 |
| `acos` | `v.acos()` | 反余弦 |
| `acosh` | `v.acosh()` | 反双曲余弦 |
| `asin` | `v.asin()` | 反正弦 |
| `asinh` | `v.asinh()` | 反双曲正弦 |
| `atan` | `v.atan()` | 反正切（单参数） |
| `atanh` | `v.atanh()` | 反双曲正切 |
| `ceil` | `v.ceil()` | 向上取整 |
| `cos` | `v.cos()` | 余弦 |
| `cosh` | `v.cosh()` | 双曲余弦 |
| `degrees` | `v.to_degrees()` | 弧度转角度 |
| `exp` | `v.exp()` | 指数函数 eˣ |
| `exp2` | `v.exp2()` | 底数为 2 的指数 2ˣ |
| `floor` | `v.floor()` | 向下取整 |
| `fract` | `v.fract()` | 小数部分 |
| `log` | `v.ln()` | 自然对数 ln(x) |
| `log2` | `v.log2()` | 底数为 2 的对数 |
| `radians` | `v.to_radians()` | 角度转弧度 |
| `round` | `v.round()` | 四舍五入 |
| `sign` | 符号函数 | 返回 -1/0/1 |
| `sin` | `v.sin()` | 正弦 |
| `sinh` | `v.sinh()` | 双曲正弦 |
| `sqrt` | `v.sqrt()` | 平方根 |
| `tan` | `v.tan()` | 正切 |
| `tanh` | `v.tanh()` | 双曲正切 |
| `trunc` | `v.trunc()` | 截断取整 |

实现模式：
```rust
native.add_method(heap, math, id_lut!(sin), script_args!(x = 0.0), |vm, args| {
    let x_val = vm.bx.heap.value(args, id!(x).into(), trap);
    NumericValue::from_script_value_vm(vm, x_val)
        .map_f32(|v| v.sin())
        .to_script_value_vm(vm)
});
```

### `inverseSqrt`（第 297–310 行）
`1.0 / sqrt(x)`，使用 `v.sqrt().recip()` 实现。

### `inverse`（第 312–328 行）
矩阵求逆。仅支持 `NumericValue::Mat4` 类型，使用 `makepad_math::Mat4f::invert()` 计算。

### `length`（第 330–343 行）
向量长度（模）。返回标量 f64，通过 `NumericValue::length()` 计算。

### `dot`（第 688–707 行）
向量点积。返回标量 f64，通过 `NumericValue::dot()` 计算。

### `normalize`（第 709–722 行）
向量归一化。返回同类型单位向量，通过 `NumericValue::normalize()` 计算。

### `cross`（第 724–742 行）
向量叉积。仅对 Vec3f 有效，通过 `NumericValue::cross()` 计算。

### `distance`（第 668–687 行）
两点间距离。计算差向量的长度。

### `max` / `min`（第 744–780 行）
逐元素最大值/最小值。使用 `zip_f32` 模式处理向量。

### `pow`（第 782–799 行）
幂运算 xʸ。使用 `a.powf(b)`。

### `modf`（第 801–819 行）
浮点取模（fmod = float mod）。使用 `a % b` 运算符。

### `step`（第 821–845 行）
阶跃函数：`x < edge ? 0.0 : 1.0`。支持标量 edge + 向量 x 的混合模式（标量广播）。若 edge 为标量调用 `step_scalar`，否则逐元素比较。

### `clamp`（第 849–878 行）
将值限制在 `[min, max]` 范围。支持标量 min/max + 向量 x 的混合模式（标量广播），否则调用 `zip_f32` 组合 `max + min`。

### `mix`（第 880–950 行）
线性插值 `x * (1-a) + y * a`（HLSL 的 lerp 等价）。支持标量 alpha 广播和逐分量 alpha。当 alpha 为向量时，按类型匹配分发到 Vec2/3/4/Color 的逐分量插值。回退到标量 alpha 模式时使用默认值 0.5。

### `smoothstep`（第 952–1042 行）
平滑阶跃函数：在 `[e0, e1]` 区间内使用 Hermite 插值 `t²(3-2t)`。支持标量 edge 广播和逐分量模式。逐分量时对 Vec2/3/4/Color 分别处理。

### `fma`（第 1044–1069 行）
融合乘加：`a * b + c`。使用两次 `zip_f32` 链式操作。

### 着色器专属函数——运行时占位实现

#### `dFdx` / `dFdy`（第 520–549 行）
屏幕空间偏导数。在脚本运行时不可计算，返回与输入类型一致的全零值（`zero_like()`）。

#### `discard`（第 552–558 行）
片段丢弃。在脚本运行时无操作，返回 NIL。

### 位转换函数

#### `asuint`（第 562–590 行）
将浮点或整数的位模式重新解释为无符号整数：
- `u32` → 原值
- `i32` → 转换为 `u32`
- `f32/f16` → `to_bits()` 取位模式

#### `asint`（第 591–619 行）
重新解释为有符号整数。`f32` 和 `f16` 通过 `to_bits()` 再转换为 i32。

#### `asfloat`（第 620–645 行）
重新解释为浮点数。`u32` 通过 `f32::from_bits()`，`i32` 先转换为 u32。

---

## `type_table_builtin`（第 1072–1553 行）

着色器编译期的类型推导和错误检查函数。VM 字节码生成着色器代码时，调用此函数确定内置数学函数的返回值类型，并验证参数类型是否正确。

### 类型速记符（第 1081–1113 行）
定义了一系列辅助类型和检查闭包：
- `is_float(t)`: t 是 f32 或 f16
- `is_int(t)`: t 是 u32 或 i32
- `is_vec_float(t)`: t 是 vec*f 或 vec*h
- `is_vec_int(t)`: t 是 vec*u 或 vec*i
- `is_any_float(t)` / `is_any_int(t)`: 标量或向量的合并检查

### 位转换类型检查（第 1115–1178 行）
- **`asuint`**: 接收 f32/f16/u32，返回 u32
- **`asint`**: 接收 f32/f16/i32，返回 i32
- **`asfloat`**: 接收 u32/i32/f32，返回 f32

### 浮点一元函数（第 1180–1240 行）
`acos~dFdy` 等 24 个函数类型规则：
- 必须恰好 1 个参数
- 必须是浮点或浮点向量类型
- 返回类型与输入类型一致
- `inverse` 特殊处理：仅接受 mat4x4f

### `discard`（第 1242–1252 行）
不接受参数，返回 void。

### `length`（第 1253–1278 行）
- 接受浮点向量类型
- vec*f → f32，vec*h → f16，标量 → 原类型

### `normalize`（第 1280–1299 行）
- 必须恰好 1 个浮点向量参数
- 返回类型与输入一致

### `abs` / `sign`（第 1301–1322 行）
接受浮点或整数（标量或向量），返回类型与输入一致。

### 浮点二元函数（第 1324–1346 行）
`atan2`、`pow`、`modf`：
- 必须 2 个同类型浮点参数
- 返回类型与参数一致

### `step`（第 1347–1373 行）
- 支持 edge 和 x 同类型
- 支持标量 edge + 向量 x（广播），如 `step(f32, vec2f)`

### `distance` / `dot`（第 1374–1402 行）
- 接受 2 个同类型浮点向量
- vec*f → f32，vec*h → f16，标量 → 标量

### `cross`（第 1404–1429 行）
- 必须 2 个同类型 vec3 参数
- vec3f → vec3f，vec3h → vec3h

### `max` / `min`（第 1431–1453 行）
接受浮点或整数（标量或向量），两个参数类型必须一致。

### `mix`（第 1455–1478 行）
- 3 个参数：x、y、alpha
- x 和 y 必须是同类型浮点值
- alpha 可以是同类型，或标量（与向量 x 组合时）

### `smoothstep` / `fma`（第 1479–1502 行）
- 3 个参数必须全部同类型浮点
- 类型一致即可（a、b、c 同类型）

### `clamp`（第 1504–1525 行）
- 3 个参数必须全部同类型
- 支持浮点和整数类型（标量或向量）

### `depth_clip`（第 1526–1547 行）
Makepad 自定义函数，用于深度裁剪：
- 签名 `(vec4f, vec4f, float) → vec4f`
- 第一个参数是世界坐标，第二个是颜色，第三个是裁剪值

### 未知函数（第 1548–1551 行）
所有未匹配的内建函数名 → 报错 "unknown shader builtin function"。

---

## 文件职责总结

`shader_builtins.rs` 涵盖着色器内置函数的完整生命周期：

1. **运行时实现**（`define_shader_builtins`）:
   - 将 40+ 个数学函数注册为 VM 原生方法
   - 支持标量和向量类型的统一处理（通过 `NumericValue`）
   - 着色器专属函数（偏导数、discard）的 CPU 占位实现
   - 位级转换函数（asuint/asint/asfloat）

2. **编译期类型检查**（`type_table_builtin`）:
   - 为每个内置函数定义严格的类型签名
   - 参数数量验证
   - 类型一致性检查（浮点 vs 整数、标量 vs 向量）
   - 特殊签名处理（标量广播、混合类型参数）
   - 错误消息中提供类型名格式化

3. **前后端协调**: 运行时行为的语义与着色器编译期的类型推导保持一致，确保脚本代码在 CPU 和 GPU 上的行为一致。
