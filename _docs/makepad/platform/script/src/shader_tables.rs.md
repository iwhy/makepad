# shader_tables.rs — 着色器类型提升与算术类型推导表

**文件路径**: `platform/script/src/shader_tables.rs`  
**行数**: 651 行  
**作用**: 定义着色器编译器的类型提升/降级表（type promotion/demotion tables）。这些纯函数接收操作数的 `ShaderType` 和内置类型系统，返回操作结果的类型。覆盖一元负号、浮点算术、整数算术、逻辑运算、比较运算、if-else 类型合并、以及复合元素类型的提取。

---

## 类型提升表的设计哲学

着色器类型系统需要处理两种类型的类型匹配：
1. **抽象类型**（`AbstractInt`/`AbstractFloat`）：来自字面量，尚未确定具体宽度
2. **具体类型**（`Pod(f32/f16/u32/i32/vec*/mat*)`）：已确定的着色器类型

类型提升的规则是：
- 抽象类型可以提升到任一兼容的具体类型
- 具体类型之间必须完全匹配
- 向量/矩阵运算涉及维度兼容性检查（如 vec2f × mat2x2f → vec2f）
- 不兼容的组合返回 `ShaderType::Error` 并设置陷阱错误

---

## `type_table_neg`（第 8–34 行）

一元取负操作的类型表。支持的具体类型：
- **抽象类型**: AbstractInt 和 AbstractFloat 维持原抽象类型
- **标量**: f32、f16、i32（不支持 u32，因无符号数取负未定义）
- **向量**: vec2f~vec4f、vec2h~vec4h、vec2i~vec4i（不支持 vec*u）

不兼容类型返回 `ShaderType::Error` 并报错 "opcode not defined for type"。

---

## `type_table_float_arithmetic`（第 36–373 行）

浮点类型算术操作（+、-、\*、/、%）的类型推导表。这是最有意义最复杂的类型表，覆盖了标量-向量混合运算和向量-矩阵乘法。

### 抽象类型交互

**AbstractFloat × rhs**:
- AbstractFloat → AbstractFloat（保留抽象）
- AbstractInt → AbstractFloat（整数提升为浮点）
- 具体 f32/f16 → 对应具体类型
- 具体 vec*f/vec*h → 对应向量类型
- 具体 mat* → 对应矩阵类型（scalar × matrix）

**AbstractInt × rhs**:
- AbstractFloat → AbstractFloat（float 优先）
- AbstractInt → AbstractInt
- 具体 u32/i32 → 对应类型
- 具体 vec*f/vec*h → 对应向量类型（abstract int 广播到向量）
- 具体 vec*u/vec*i → 对应向量类型
- 具体 mat* → 对应矩阵类型

### 标量具体类型交互

**f32 × rhs**:
- AbstractFloat/AbstractInt → f32
- f32 → f32
- vec2f/3f/4f → 对应向量类型（标量广播）
- mat* → 对应矩阵类型（标量 × 矩阵）

**f16 × rhs**:
- 类似 f32 但限于 f16 家族（vec2h/3h/4h）

**u32 × rhs**:
- AbstractFloat/AbstractInt → u32
- u32 → u32
- vec2u/3u/4u → 对应向量（注意：u32 与 AbstractFloat 交互时仍保持 u32，不支持隐式浮点提升）

**i32 × rhs**:
- 类似 u32，涉及 vec2i/3i/4i

### 向量-矩阵乘法（第 196–366 行）

向量 × 矩阵的维度推导规则，遵循线性代数约定 `vecR × matRxC = vecC`：

| lhs | rhs | 结果 | 规则 |
|-----|-----|------|------|
| vec2f | mat2x2f | vec2f | 矩阵 × 向量，同维度 |
| vec2f | mat3x2f | vec3f | 2D 向量 × 2×3 矩阵 → 3D 向量 |
| vec2f | mat4x2f | vec4f | 2D 向量 × 2×4 矩阵 → 4D 向量 |
| vec3f | mat2x3f | vec2f | 3D 向量 × 3×2 矩阵 → 2D 向量 |
| vec3f | mat3x3f | vec3f | 同维度 |
| vec3f | mat4x3f | vec4f | 3D → 4D 升维 |
| vec4f | mat2x4f | vec2f | 4D → 2D 降维 |
| vec4f | mat3x4f | vec3f | 4D → 3D |
| vec4f | mat4x4f | vec4f | 同维度 |

矩阵 × 标量返回原矩阵类型。

**矩阵 × 矩阵**:
- mat2x2f × mat2x2f → mat2x2f
- mat3x3f × mat3x3f → mat3x3f
- mat4x4f × mat4x4f → mat4x4f

错误路径标为 `ShaderType::Error(NIL)` 并报错 "no wgsl conversion"。

---

## `type_table_int_arithmetic`（第 375–433 行）

整数算术操作（&、|、^、<<、>>、%）的类型推导表。相比浮点表，限制更多：

- **AbstractFloat × rhs**: 总是 Error（整数与浮点不兼容）
- **AbstractInt × rhs**: AbstractInt 或具体 u32/i32
- **u32/i32 × rhs**: 仅同类型或抽象类型
- **vec*u/vec*i × rhs**: 仅向量同类型之间（vec2u × vec2u → vec2u）
- 不支持矩阵运算
- 不支持 vec*u × vec*i 混合

---

## `type_table_logic`（第 435–453 行）

逻辑操作（`&&`、`||`、`!`）的类型推导表。

- 只接受 `Pod(bool) × Pod(bool) → Pod(bool)`
- 不支持和整数的位运算混淆

---

## `type_table_eq`（第 455–560 行）

比较操作（`==`、`!=`、`<`、`>`、`<=`、`>=`）的类型推导表。

**标量比较**:
- AbstractFloat/AbstractInt × AbstractFloat/AbstractInt → `bool`
- Abstract × 具体类型（f32/f16/u32/i32）→ `bool`
- 具体类型 × 抽象或同类型 → `bool`

**向量比较**（逐分量比较）:
- vec2f/2h/2u/2i × 同类型 → `vec2b`（逐分量布尔向量）
- vec3f/3h/3u/3i × 同类型 → `vec3b`
- vec4f/4h/4u/4i × 同类型 → `vec4b`
- bool × bool → `bool`

向量比较返回布尔向量类型，用于逐分量掩码操作。

---

## `type_table_if_else`（第 562–611 行）

if-else 表达式的类型合并表。确定 `if cond { expr_a } else { expr_b }` 中两个分支的公共类型。

**规则**:
- AbstractFloat × AbstractFloat → AbstractFloat
- AbstractFloat × AbstractInt → 提升为 AbstractFloat
- AbstractFloat × 具体 f32/f16 → 具体类型
- AbstractInt × AbstractFloat → AbstractFloat（float 优先）
- AbstractInt × AbstractInt → AbstractInt
- AbstractInt × 具体 f32/f16/i32/u32 → 具体类型
- 具体类型 × 抽象类型（兼容时）→ 具体类型
- 具体类型 × 同类型 → 同类型
- 其他组合 → `ShaderType::Error`，报告 "if-else type mismatch"

---

## `type_table_elem_type`（第 613–651 行）

获取复合类型的元素类型，用于索引操作和数组元素访问。

**数组类型**:
- `FixedArray { ty }` → ty.self_ref（元素类型自身）
- `VariableArray { ty }` → ty.self_ref

**向量类型**:
- Vec2f/3f/4f → f32
- Vec2h/3h/4h → f16
- Vec2u/3u/4u → u32
- Vec2i/3i/4i → i32
- Vec2b/3b/4b → bool

**矩阵类型**:
- Mat* x2f（列向量为 vec2）→ vec2f
- Mat* x3f（列向量为 vec3）→ vec3f
- Mat* x4f（列向量为 vec4）→ vec4f

**其他类型** → `None`

---

## 文件职责总结

`shader_tables.rs` 是 Makepad 着色器编译器类型系统的"算术逻辑单元"，职责包括：

1. **一元运算类型检查**（neg）：验证取负操作的类型合法性
2. **二元算术类型提升**（float/int）：处理标量-向量混合运算、向量-矩阵乘法维度的自动推导
3. **逻辑运算类型检查**：确保 bool 类型专用操作符的类型安全
4. **比较运算类型推导**：标量比较返回 bool，向量比较返回逐分量布尔向量
5. **控制流类型合并**（if-else）：确定分支表达式的公共类型，处理抽象类型降级
6. **复合类型元素类型提取**：为数组索引、向量/矩阵分量访问提供元素类型信息

所有类型表函数遵循同一模式：匹配成功返回新类型，匹配失败返回 `ShaderType::Error` 并设置陷阱错误。
