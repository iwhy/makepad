# `mod_pod.rs` — POD 类型注册模块

## 概述

本文件定义了 Splash 脚本运行时的 POD（Plain Old Data）类型系统初始化函数。POD 类型是 GPU 友好的紧凑内存布局类型，用于着色器接口、数学运算和序列化。模块通过 `define_pod_module` 在 `mod.pod` 空间下注册所有原子、向量和矩阵类型，并提供 `mix` 方法在 POD 值之间进行线性插值。

---

## 宏

### `script_pod_def!`

```rust
macro_rules! script_pod_def {
    ($heap:expr, $pod: expr, $ty: ident, $id:ident, $pod_ty:expr, $pod_def:expr )
```

**实现逻辑**:
1. 使用 `$heap.new_with_proto` 创建一个以 `$ty`（如 `f32`）的 LiveId 为原型的脚本对象。
2. 调用 `$heap.new_pod_type($pod_ty, $pod_def)` 创建一个新的 POD 类型描述符。
3. `set_object_storage_vec2` 设置对象内部存储方式为 Vec2（变长键值对存储）。
4. `set_object_pod_type` 将该对象与刚创建的 POD 类型关联起来。
5. `set_value_def` 在 `$pod` 模块对象的 `$id` 键名下注册该 POD 类型对象。
6. 返回 `ScriptPodType` 句柄供后续索引。

---

## 核心数据结构

### `ScriptPodBuiltins`

存储所有 30 种内置 POD 类型的 `ScriptPodType` 句柄，分为四类：

| 类别 | 类型 |
|------|------|
| **特殊** | `void`, `struct`, `array` |
| **原子** | `bool`, `f32`, `f16`, `u32`, `i32`, `atomic_u32`, `atomic_i32` |
| **向量** | `vec2f/vec3f/vec4f`, `vec2h/vec3h/vec4h`, `vec2u/vec3u/vec4u`, `vec2i/vec3i/vec4i`, `vec2b/vec3b/vec4b` |
| **矩阵** | `mat2x2f` 到 `mat4x4f` (9 种) |

---

## `define_pod_module(heap, native) → ScriptPodBuiltins`

### 模块创建

- 调用 `heap.new_module(id!(pod))` 创建 `mod.pod` 模块对象。

### 特殊类型注册

- **`pod_void`** (`ScriptPodTy::Void`): 无类型标记。断言 `pod_void == ScriptPodType::VOID` 验证常量正确性。
- **`pod_struct`** (`ScriptPodTy::UndefinedStruct`): 结构体占位符，在结构体定义完成前使用。
- **`pod_array`** (`ScriptPodTy::ArrayBuilder`): 数组构造器类型，用于动态创建 POD 数组。

### 原子类型注册

每个原子类型调用 `heap.pod_def_atom`，参数为：模块对象、类型名、可选别名（alias）、`ScriptPodTy` 枚举、LiveId 和默认值：

| 类型 | 别名 | 默认值 |
|------|------|--------|
| `bool` | 无 | `false` |
| `f32` | `float` | `0.0f32` |
| `f16` | 无 | `0.0f16` |
| `u32` | `uint` | `0u32` |
| `i32` | `int` | `0i32` |
| `atomic_u32` | 无 | `0u32` |
| `atomic_i32` | 无 | `0i32` |

**别名机制**: `f32` 的别名为 `float`，`u32` 的别名为 `uint`，`i32` 的别名为 `int`，这使得脚本中可以使用 `pod.float` 访问 `pod.f32`。

### 向量类型注册

每个向量类型调用 `heap.pod_def_vec`。可选别名使 `pod.vec2` 可以访问 `pod.vec2f`。完整的九对向量类型覆盖了 `float/half/uint/int/byte` × `2/3/4` 的笛卡尔积。

### 矩阵类型注册

九个矩阵类型调用 `heap.pod_def_mat`，覆盖 `2×2` 到 `4×4` 的所有组合，全部基于 `f32` 元素。

---

## POD 方法：`mix`

### `pod_value.mix(other, a) → pod_value`

**注册**: `native.add_type_method(heap, ScriptValueType::REDUX_POD, id!(mix), ...)`

**实现逻辑**:
1. 从参数中提取 `self`、`other`（目标值）和 `a`（插值因子）。
2. 将 `self` 和 `other` 通过 `NumericValue::from_script_value_heap` 转换为数值表示。
3. 根据 `a` 的类型选择插值路径：
   - **标量路径** (`a.as_f64()` 成功): 调用 `self_nv.mix_scalar(other_nv, a_f)`，对所有分量使用相同的 `a`。
   - **分量路径**: 将 `a` 也转为 `NumericValue`，调用 `self_nv.mix_componentwise(other_nv, a_nv)`，允许每个分量有不同的 `a`（例如在 `vec4f` 上按 rgba 通道独立插值）。
4. 结果通过 `to_script_value_heap` 转换回 `ScriptValue` 返回。

---

## `value_to_exact_type(val) → Option<ScriptPodType>`

**实现逻辑**:
1. 根据 `ScriptValue` 的内部类型枚举，判断精确对应的 POD 类型：
   - `f64` / `f32` → `pod_f32`
   - `u40` / `u32` → `pod_u32`
   - `i32` → `pod_i32`
   - `f16` → `pod_f16`
   - `bool` → `pod_bool`
2. 不匹配时返回 `None`。

---

## 设计要点

- **完整的内置类型覆盖**: 30 种 POD 类型覆盖了 GPU 编程中所有常见的标量、向量和矩阵格式。
- **别名系统**: `float`/`uint`/`int` 别名提供了更符合脚本开发者习惯的命名。
- **双路径 mix**: `mix_scalar` 和 `mix_componentwise` 的区分使单一 `mix` 方法既能用于简单的标量插值（如 `a.mix(b, 0.5)`），也能用于复杂的逐分量插值（如颜色混合）。
