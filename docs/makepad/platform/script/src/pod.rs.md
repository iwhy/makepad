# `pod.rs` — 纯数据值类型系统：`ScriptPod` 与 WGSL 内存布局

## 文件位置
`platform/script/src/pod.rs` (583 行)

## 概述
POD（Plain Old Data）系统为 Makepad 脚本引擎提供了**类型化的、内存布局精确的**纯数据值表示，直接映射到 WGSL 着色器的数据布局规范（std140 对齐规则）。

---

## 核心类型

### `ScriptPodField` — POD 结构体字段
```rust
pub struct ScriptPodField {
    pub name: LiveId,               // 字段名
    pub default: ScriptValue,       // 默认值
    pub ty: ScriptPodTypeInline,    // 字段类型
}
```

### `ScriptPodEnum` — POD 枚举变体
```rust
pub struct ScriptPodEnum {
    pub name: LiveId,
    pub variant: ScriptPodEnumVariant,
}
```

### `ScriptPodEnumVariant` — 枚举变体类型
```rust
pub enum ScriptPodEnumVariant {
    Bare,                                    // 无字段（如 C 风格枚举）
    Tuple { items: Vec<ScriptPodTypeInline> },// 元组体变体
    Named { fields: Vec<ScriptPodField> },    // 命名字段体变体
}
```

### `ScriptPodTypeData` — 类型完整描述
```rust
pub struct ScriptPodTypeData {
    pub name: Option<LiveId>,       // 可选类型名称
    pub object: ScriptObject,       // 关联的脚本对象（类型元数据）
    pub default: ScriptValue,       // 默认值
    pub ty: ScriptPodTy,            // 类型定义
}
```

### `ScriptPodTypeInline` — 内联类型引用
```rust
pub struct ScriptPodTypeInline {
    pub self_ref: ScriptPodType,  // 自引用（类型索引）
    pub data: ScriptPodTypeData,  // 类型数据
}
```
`self_ref` 允许类型引用自身（递归类型），同时 `data` 提供完整类型描述的快速访问。

---

### `ScriptPodVec` — 向量类型枚举
涵盖所有 WGSL 向量类型，按元素类型分组：

| 类型 | 精度 | 元素 |
|------|------|------|
| `Vec2f/Vec3f/Vec4f` | f32 | 32 位浮点 |
| `Vec2h/Vec3h/Vec4h` | f16 | 16 位半精度浮点 |
| `Vec2u/Vec3u/Vec4u` | u32 | 32 位无符号整型 |
| `Vec2i/Vec3i/Vec4i` | i32 | 32 位有符号整型 |
| `Vec2b/Vec3b/Vec4b` | bool | 布尔值 |

#### 方法

**`elem_size()`**: 返回元素字节大小。半精度（h）返回 2，其余返回 4。

**`elem_ty()`**: 返回元素的基础 POD 类型（F32/F16/U32/I32/Bool）。

**`name()`**: 返回对应的 `LiveId`，用于脚本中类型名称匹配，如 `id!(vec2f)`。

**`swizzle_type(lanes, builtin)`**: 根据 swizzle 操作的道数返回对应的向量类型。例如 `Vec4f.swizzle_type(2, builtin)` 返回 `builtin.pod_vec2f`，`Vec4f.swizzle_type(1, builtin)` 返回 `builtin.pod_f32`。这是一个巨大的 match 表，用于编译时/运行时 swizzle 表达式的类型推导。

**`builtin(builtin)`**: 返回该向量类型在 `ScriptPodBuiltins` 中的内置类型引用。

**`dims()`**: 返回向量维度（2/3/4）。

**`align_of()`**: 返回对齐要求（字节），遵循 WGSL std140 规则：`vec2` 对齐到 8 字节，`vec3`/`vec4` 对齐到 16 字节。

**`size_of()`**: 返回字节大小：`vec2=8`、`vec3=12`、`vec4=16`（半精度折半）。

---

### `ScriptPodMat` — 矩阵类型枚举

| 方法 | 说明 |
|------|------|
| `elem_size()` | 矩阵元素总为 4 字节（f32） |
| `name()` | 返回矩阵类型 LiveId |
| `builtin(builtin)` | 返回内置类型引用 |
| `dim()` | 总元素数（`rows * cols`） |
| `dims()` | 返回 `(cols, rows)` 元组 |
| `align_of()` | 对齐：列数 2 对齐 8 字节，列数 3/4 对齐 16 字节 |
| `size_of()` | 大小：`cols * rows * 4`，但按列对齐规则上取整（每列对齐到 16 字节） |

注意矩阵布局：WGSL 矩阵是列优先（column-major），文档中 `Mat2x3f` 表示 2 列 3 行，即 `Mat<cols=x, rows=y>`。

---

### `ScriptPodTy` — 核心类型系统枚举
```rust
pub enum ScriptPodTy {
    Void,               // 空类型
    ArrayBuilder,       // 数组构建器（特殊中间状态）
    UndefinedStruct,    // 未定义结构体（前向引用占位）
    F32, F16, U32, I32, Bool,  // 标量类型
    AtomicU32, AtomicI32,      // WGSL 原子类型
    Vec(ScriptPodVec),          // 向量
    Mat(ScriptPodMat),          // 矩阵
    Struct { align_of, size_of, fields },    // 结构体
    Enum { align_of, size_of, variants },    // 枚举
    FixedArray { align_of, size_of, len, ty }, // 定长数组
    VariableArray { align_of, ty },           // 变长数组（运行时数组）
}
```

#### `is_number()` — 是否为数值标量
检查 `F32 | F16 | U32 | I32`。注意 `Bool` 和 `Atomic` 不计为数值。

#### `is_float_type()` — 是否为浮点类型
递归检查：标量 F32/F16、向量（元素类型为浮点）、矩阵（总是浮点）。用于确定是否需要浮点运算指令。

#### `calculate_struct_layout(fields)` — 计算结构体布局
实现 WGSL std140 对齐算法：
1. **计算最大对齐**：所有字段 `align_of` 的最大值。
2. **计算偏移**：遍历字段，对每个字段：
   - 当前偏移对其对齐要求取模，如果不为 0，增加填充字节对齐。
   - 增加该字段的 `size_of`。
3. **最终对齐**：总大小对结构体对齐取模取整。
4. 返回 `(align_of, total_size)`。

#### `new_struct(fields)` — 创建结构体类型
调用 `calculate_struct_layout` 自动计算布局后构造 `ScriptPodTy::Struct` 变体。

#### `align_of()` — 对齐要求
递归计算所有变体的对齐：
- 标量：F32/U32/I32/Bool/Atomic = 4，F16 = 2
- Void/ArrayBuilder/UndefinedStruct = 0
- Vec/Mat：委托对应的 align_of
- Struct/Enum/FixedArray/VariableArray：返回预计算的 `align_of`

#### `size_of()` — 字节大小
- 标量：同上（Bool 占 4 字节）
- Vec/Mat：委托对应的 size_of
- Struct/Enum/FixedArray：返回预计算的 `size_of`
- VariableArray：返回 0（变长数组不计算固定大小）

#### `slots()` — 按 f32 槽位数
`(size_of() + 3) / 4`，将字节大小上取整到 4 字节槽位数。用于着色器属性/统一缓冲区布局计算。

---

### `ScriptPodOffset` — POD 字段偏移
```rust
pub struct ScriptPodOffset {
    pub offset_of: usize,   // 字节偏移
    pub field_index: usize, // 字段索引
}
```
双重索引：`offset_of` 用于直接内存访问，`field_index` 用于类型元数据查找。

---

### `ScriptPodTag(u64)` — POD 标签
与 `ScriptObjectTag`/`ScriptArrayTag` 类似的位标志模型，但移位到 60 位（避免与低 40 位的引用数据冲突）：

| 标志 | 用途 |
|------|------|
| `MARK (0x1 << 60)` | GC 标记 |
| `ALLOCED (0x2 << 60)` | 已分配 |
| `ARRAY_BUILDER (0x4 << 60)` | 数组构建器模式 |
| `STATIC (0x8 << 60)` | 静态不可回收 |

#### `set_array_builder(ty)` / `as_array_builder()` — 数组构建器
`set_array_builder` 设置 `ARRAY_BUILDER` 位并将 `ty.index` 写入低 32 位。`as_array_builder` 读取并返回。这是变长数组构建阶段的中间状态标记。

---

### `ScriptPodData` — POD 数据容器
```rust
pub struct ScriptPodData {
    pub tag: ScriptPodTag,      // GC 标签
    pub ty: ScriptPodType,      // 类型索引
    pub data: Vec<u32>,         // 原始二进制数据（按 u32 对齐）
}
```

`data: Vec<u32>` 是核心设计：POD 数据以 `u32` 为单位存储，这是 GPU 统一的缓冲区格式。所有标量、向量、矩阵、结构体最终都序列化到此 `Vec<u32>` 中。例如：
- `f32` 占 1 个 `u32`
- `vec2f` 占 2 个 `u32`
- `vec3f` 占 3 个 `u32`
- `mat4x4f` 占 16 个 `u32`

#### `clear()` — 清空
清空 `tag` 和 `data`。
