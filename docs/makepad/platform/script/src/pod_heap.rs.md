# `pod_heap.rs` — POD 堆操作

## 文件位置
`platform/script/src/pod_heap.rs` (2357 行)

## 概述
`pod_heap.rs` 在 `ScriptHeap` 上实现了所有与 **POD (Plain Old Data)**相关的操作。POD 是 Makepad 脚本引擎中与 GPU 着色器直接交互的**紧凑二进制数据**类型，包括：
- **POD 类型定义**（标量、向量、矩阵、结构体、定长/变长数组、枚举）
- **POD 类型注册**（`pod_def_atom` / `pod_def_vec` / `pod_def_mat`）
- **POD 分配**（`new_pod`）
- **POD 字段写入**（`pod_pop_to_me` / `pod_write_field` / `pod_write_vec`）
- **POD 字段读取**（`pod_read_field` / `pod_array_index`）
- **向量 swizzle 操作**（`.xyzw` / `.rgba` 等重排访问）
- **类型推导与类型映射**（Rust TypeId → ScriptPodType）
- **调试输出**（`pod_debug`）
- **f16/f32 互转工具函数**

---

## POD 类型系统 (`ScriptPodTy`)

POD 类型系统支持以下类型：

| 类型变体 | 示例 | 说明 |
|---|---|---|
| `F32` / `F16` / `U32` / `I32` / `Bool` | `f32`, `u32` | 标量类型 |
| `Vec(Vec2/3/4[f/h/u/i/b])` | `vec2f`, `vec3h` | 向量类型（f=f32, h=f16, u=u32, i=i32, b=bool） |
| `Mat(Mat4x4f)` | `mat4x4f` | 矩阵类型（目前仅 4x4 f32） |
| `Struct{fields, align_of, size_of}` | 自定义结构体 | 带字段名和布局的结构体 |
| `FixedArray{ty, len, size_of, align_of}` | `[f32; 4]` | 定长数组 |
| `VariableArray{ty, align_of}` | `[f32]` | 变长数组 |
| `ArrayBuilder` | 临时构造器 | 用于构造数组内容的中间状态 |
| `Enum` | 枚举 | 枚举类型 |
| `UndefinedStruct` | - | 未定义结构体（占位） |

---

## 方法详解

### POD 类型操作

#### `pod_method(&self, ptr: ScriptPod, key, trap) -> ScriptValue`
**获取 POD 的方法**。POD 类型对象上的方法实际存储在 `pod_type.object` 上，所以此方法委托给 `self.value(pod_ty.object, key, trap)` 进行原型链查找。这意味着 POD 可以通过其类型对象获得方法。

#### `new_pod_type(&mut self, object, name, ty, default) -> ScriptPodType`
**创建新的 POD 类型描述符**。尝试从 `pod_types_free` 回收空闲槽位，否则推送到 `pod_types` 向量。返回 `ScriptPodType{index}`。

#### `new_pod_array_type(&mut self, ty, default) -> ScriptPodType`
**为数组元素创建/查找 POD 类型**。遍历已有 `pod_types`，如果找到匹配的 `ty` 则复用，否则创建一个新的。用于 `VariableArray` 和 `FixedArray` 的元素类型绑定。

#### `pod_type(&self, ty: ScriptValue) -> Option<ScriptPodType>`
**从 ScriptValue 提取 POD 类型**。如果值是对象且其 tag 中包含 `pod_type` 标记，则返回该标记中的 `ScriptPodType`。

#### `pod_type_ref(&self, ty: ScriptPodType) -> &ScriptPodTypeData`
**获取 POD 类型数据引用**。

#### `pod_data(&self, pod: ScriptPod) -> (&ScriptPodTypeData, &[u32])`
**获取 POD 数据和类型信息**。返回 (类型数据, u32 数据切片) 对。

#### `pod_type_name / pod_type_name_set / pod_type_name_if_not_set`
POD 类型名称的读/写/条件写入。

#### `type_id_to_pod_type(&self, type_id, builtins) -> Option<ScriptPodType>`
**Rust TypeId → ScriptPodType 映射**。这是脚本引擎与 Rust 类型系统之间的桥梁：
- 直接映射：`f32 → pod_f32`, `u32 → pod_u32`, `i32 → pod_i32`, `bool → pod_bool`
- 向量映射：`Vec2f → pod_vec2f`, 等
- 矩阵映射：`Mat4f → pod_mat4x4f`
- `Quat` 复用 `pod_vec4f`（布局相同）
- 如果类型已注册且标记了 `is_repr_u32_enum` → `pod_u32`
- 如果类型的原型对象有 `pod_type` tag → 返回该类型

#### `pod_type_inline(&self, val, builtins) -> Option<ScriptPodTypeInline>`
**从值内联推导 POD 类型**。检查值的具体类型（对象上的 pod_type tag、f64、f32、u32、i32、f16、bool），返回包含 `self_ref` 和克隆的 `ScriptPodTypeData` 的结构体。

#### `finalize_maybe_pod_type(&mut self, ptr, builtins, trap)`
**最终确定可能的 POD 类型**。如果对象被标记为 `is_pod_type()`，根据其原型 ID 决定如何构建类型：
- `pod_array` → 读取 kv 构建 `VariableArray`（单元素定义）或 `FixedArray`（[长度, 元素类型] 定义）
- `pod_struct` → 遍历字段，非函数值作为 `ScriptPodField` 收集，函数值作为方法收集。调用 `ScriptPodTy::new_struct` 计算对齐和大小
- 其他 → 不可扩展

#### `pod_def_atom(&mut self, pod_module, name, alias, ty, helper_name, default) -> ScriptPodType`
**定义原子 POD 类型**（标量、枚举等）。创建一个类型对象并注册到模块中，可选别名。

#### `pod_def_vec(&mut self, pod_module, name, alias, builtin) -> ScriptPodType`
**定义向量 POD 类型**。创建类型对象，设置为 `Vec` 类型，标记 notproto + freeze，注册到模块。

#### `pod_def_mat(&mut self, pod_module, name, builtin) -> ScriptPodType`
**定义矩阵 POD 类型**。与 `pod_def_vec` 类似，但对矩阵类型。

---

### POD 分配与写入

#### `new_pod(&mut self, ty: ScriptPodType) -> ScriptPod`
**分配新 POD**。从 `pods_free` 回收槽位或 push 新槽位。计算 `ty.size_of().next_multiple_of(4) >> 2` 来分配足够的 u32 槽位，并清零数据。

#### `set_pod_field(&self, pod, field, value, trap) -> ScriptValue`
**设置 POD 字段**（当前为 stub 实现，仅日志输出）。

#### `pod_pop_to_me(&mut self, pod_ptr, offset, _field, value, builtins, trap)`
**顺序构造 POD 内容**。这是 POD 构造函数的核心：每次调用写入一个字段的值，更新 `ScriptPodOffset` 中的偏移量。根据 POD 类型分支处理：
- **`Struct`**：对齐偏移，写入当前字段，推进 `field_index` 和 `offset_of`
- **`ArrayBuilder`**：按元素类型对齐，写入元素，推进偏移
- **`Vec(ot)`**：调用 `pod_write_vec` 写入向量分量
- **`Mat(mt)`**：按元素写入矩阵（每个元素为 f32 数）
- **`F32/U32/I32/F16/Bool`**：单字段构造器，写入对应位的值

使用 swap-in/swap-out 模式修改数据（先 swap 出 `pod.data`，修改后再 swap 回）。

#### `pod_check_arg_total(&mut self, pod, offset, trap)`
**检查构造参数完整性**。验证 `offset.offset_of` 是否等于类型预期大小。对于 `Vec` 和 `Mat` 类型，如果只提供了一个元素（标量展开），自动填充剩余元素实现"标量广播"。

#### `pod_write_vec(&self, ot, offset_of, out_data, value, trap) -> usize`
**写入向量分量**。根据向量类型（Vec2f/h/u/i/b）和值类型（number/bool/pod）分别处理：
- 数值：写入分量值，推进 `elem_size` 个字节
- bool：仅 `Vec*b` 家族接受
- POD 值：将子 POD 的分量按类型转换后写入（向量跨类型转换的完整实现）

#### `pod_write_field(&self, field_ty, offset_of, out_data, value, trap)`
**写入结构体字段**。根据字段类型分支：
- **`Bool/U32/I32/F32/F16`**：从数值转换并写入
- **`Vec/Met`**：匹配类型后直接复制数据
- **`Struct`**：匹配类型后复制数据
- **`FixedArray`**：从 `ArrayBuilder` 写入，检查元素类型和大小
- **`VariableArray`**：从 `ArrayBuilder` 写入，动态扩展目标缓冲区

---

### POD 读取

#### `pod_array_index(&mut self, pod_ptr, index, builtins, trap) -> ScriptValue`
**POD 数组索引读取**。计算元素偏移 = `index * align_of`，进行边界检查，然后根据元素类型构造返回值：
- `Bool` → `TRUE` / `FALSE`
- `U32/U32` → `ScriptValue::from_u32`
- `I32/I32` → `ScriptValue::from_i32`
- `F32` → `ScriptValue::from_f32`
- `F16` → `ScriptValue::from_f16`（处理半字节对齐）
- `Vec/Met/FixedArray/Struct` → 分配新 POD，复制数据范围后返回

#### `pod_field_type(&self, pod_ty, field_name, builtins) -> Option<ScriptPodType>`
**获取 POD 结构体字段的类型**。遍历结构体的 `fields` 数组查找 `field_name`，返回对应字段的 `self_ref`。对向量类型支持 swizzle 匹配。

#### `pod_read_field(&mut self, pod_ptr, field, builtins, trap) -> ScriptValue`
**读取 POD 字段**。按字段名在结构体中查找：
- 对齐偏移后根据字段类型读取标量（Bool/U32/I32/F32/F16）
- 向量/矩阵/子结构体/子数组 → 分配新 POD，复制数据范围
- 向量还支持 swizzle 读取（`makepad_script_derive::pod_swizzle_vec_match!` 宏）

#### `pod_swizzle_vec1(&self, vec, data, x, trap) -> ScriptValue`
**单分量 swizzle 读取**。读取向量的第 x 个分量，返回对应的 `ScriptValue` 类型。处理边界检查。

#### `pod_swizzle_vec<const N: usize>(&mut self, vec, data, swiz, builtins, _trap) -> ScriptValue`
**多分量 swizzle 读取**。根据提取的分量数 N（2/3/4）和原始 vec 类型，确定输出 POD 类型。创建新 POD，按 swiz 索引重排数据复制。

---

### 构造器参数检查

#### `pod_check_abstract_constructor_arg(&self, pod_ty, offset, trap)`
**抽象构造器参数检查**（仅检查个数，不检查具体类型）：
- 标量类型：最多 1 个参数
- `Vec`：参数数 ≤ dims
- `Mat`：参数数 ≤ dim
- `Struct`：参数数 ≤ fields.len()
- `FixedArray`：参数数 ≤ len

#### `pod_check_constructor_arg_count(&self, pod_ty, offset, trap)`
**构造器参数个数校验**。验证 `offset.field_index` 是否达到类型要求的最小参数数：
- 标量：必须为 1
- Vec/Mat：1 或 dims
- Struct/FixedArray：必须等于字段数/长度

#### `pod_check_constructor_arg(&self, pod_ty, pod_ty_arg, offset, trap)`
**带类型的构造器参数检查**。检查参数类型是否匹配：
- 标量之间可交叉转换（f32↔f16↔u32↔i32）
- Vec 接受单分量或子向量
- Mat 接受 f32 元素
- Struct 字段类型必须严格匹配
- FixedArray/VariableArray 元素类型必须匹配
- AtomicU32/AtomicI32 不可直接构造

---

### 调试输出

#### `pod_debug(&self, out, pod_type, offset_of, data)`
**POD 调试格式化**。递归遍历 POD 结构，输出：
- 标量：`type:value`（如 `f32:3.14`）
- 向量：`vec2f(c0, c1)` 或 `vec2h(c0, c1)`（半精度浮点特殊处理）
- 矩阵：`mat4x4f([c0,c1,c2,c3], ...)` 的行主序格式
- 结构体：`struct{field1:..., field2:...}`
- 定长/变长数组：`array(0:..., 1:...)`

---

### f16/f32 转换工具

#### `f16_to_f32(h: u16) -> f32`
**半精度浮点 → 单精度浮点**。实现 IEEE 754-2008 半精度转换：
- 分离符号 (1bit)、指数 (5bit)、尾数 (10bit)
- 特殊值处理：无穷/NaN（指数 0x1F）、零/次正规数（指数 0）、正规数
- 重新偏置指数（15 → 127）并缩放尾数（10bit → 23bit）

#### `f32_to_f16(f: f32) -> u16`
**单精度浮点 → 半精度浮点**。反向转换：
- 分离符号、指数 (8bit)、尾数 (23bit)
- 特殊值处理：NaN → f16-NaN（保留尾数高位）、无穷
- 重偏执指数（127 → 15），溢出饱和到无穷，下溢到零
- 处理次正规数（渐进下溢）
- 尾数从 23bit 舍入到 10bit
