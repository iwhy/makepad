# value.rs — ScriptValue: 64-bit NaN-Boxed 值类型系统

## 概述

`value.rs` 定义了 Splash VM 的核心值类型 `ScriptValue`，它是一个 64 位 NaN-boxing 实现。所有脚本中的值（数字、字符串、对象、数组、布尔、nil、颜色、句柄、错误等）都编码在一个 `u64` 中。此外还定义了引用类型句柄（`ScriptObject`、`ScriptArray`、`ScriptString` 等），它们与堆上的 GenVec 配合实现 use-after-free 检测。

**总行数**: 1627 行

---

## NaN-Boxing 位布局方案

IEEE 754 双精度浮点数的 64 位布局是 NaN-boxing 的基础：
- **Bit 63 (符号位)**: 1
- **Bit 62-52 (指数位, 11 bits)**: `0x7FF` (全1) 表示 NaN/Inf
- **Bit 51-0 (尾数位, 52 bits)**: 有效载荷

对于普通 `f64` 数值，指数位不全为 1，所以 `0 < 0x7FF0000000000000`。

Splash VM 利用 **高 24 位**(bits 63-40) 存储类型标签，**低 40 位**(bits 39-0) 存储有效载荷：

```
63  62  61  60  59  58  57  56  55  54  53  52  51  50  49  48  47  46  45  44  43  42  41  40 | 39 ... 0
+------------------------------------------------------------------------------------------------+---------+
|  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  |                TYPE TAG (8 bits)           | PAYLOAD |
+------------------------------------------------------------------------------------------------+---------+
   <-- 指数全1 (NaN 标记) -----------> |<-- bit 51-40: 8位类型标签 -->|<-- 低 40 位有效载荷 -->
```

关键常量:
- `TYPE_MASK = 0xFFFF_FF00_0000_0000` — 提取高 24 位
- 类型标签位于 bits 47-40（8 位）
- 低 40 位用于存储索引、整数、内联字符串字节等

### 类型标签枚举 (`ScriptValueType`)

| Tag 值 | 常量 | 描述 |
|---------|------|------|
| 0 | `F64` | 普通浮点数（无 NaN 标记的实际 f64） |
| 1 | `NAN` | 追踪的 NaN（带 payload 的 NaN） |
| 2 | `F32` | 32位浮点数 |
| 3 | `F16` | 16位浮点数（存为 u32 bits） |
| 4 | `U32` | 32位无符号整数 |
| 5 | `I32` | 32位有符号整数 |
| 6 | `U40` | 40位无符号整数 |
| 7 | `BOOL` | 布尔值 |
| 8 | `NIL` | 空值 |
| 9 | `COLOR` | 颜色值 (RGBA u32) |
| 10 | `OBJECT` | 堆对象引用 |
| 11 | `ARRAY` | 堆数组引用 |
| 12 | `POD_TYPE` | POD 类型引用 |
| 13 | `POD` | POD 实例引用 |
| 14 | `REGEX` | 正则表达式引用 |
| 15 | `OPCODE` | 操作码（编译期） |
| 16 | `STRING` | 堆字符串引用 |
| 17 | `ERROR` | 错误类型基 |
| 18-24 | `INLINE_STRING_0..5` | 内联字符串（0-5 字节） |
| 24-43 | `ERR_FIRST..ERR_LAST` | 19 种分类错误 |
| 0x50-0x7F | `HANDLE_FIRST..HANDLE_LAST` | 48 种外部句柄类型 |
| 0x80 | `ID` | LiveId 标识符 |

### 类型检测优化

`is_number()` 和 `is_f64()` 利用有序性做快速范围检查：
- `is_f64()`: `self.0 <= TYPE_TRACED_NAN_MAX` — 所有小于或等于最大追踪 NaN 的值都是 f64 或普通 NaN
- `is_non_nan_number()`: `self.0 < TYPE_NAN` — 所有 f64 数值的位模式都小于 NaN 标记值
- `is_number()`: `self.0 <= TYPE_NUMBER_MAX` — 包含 F64、F32、F16、U32、I32、U40 所有数值子类型

---

## 核心类型详解

### `ScriptValue` (行 777-1555)

```rust
pub struct ScriptValue(pub u64);
```

#### 构造常量

- `NIL` — `ScriptValue(Self::TYPE_NIL)` = `0xFFFF_FF08_0000_0000`
- `TRUE` — `ScriptValue(Self::TYPE_BOOL | 1)` = `0xFFFF_FF07_0000_0001`
- `FALSE` — `ScriptValue(Self::TYPE_BOOL | 0)` = `0xFFFF_FF07_0000_0000`
- `EMPTY_STRING` — `TYPE_INLINE_STRING_0`

#### 类型查询

| 方法 | 实现逻辑 |
|------|----------|
| `value_type()` | 先检查 `is_non_nan_number()` 判断是否为普通 f64；否则提取高 24 位调用 `ScriptValueType::from_u64` |
| `is_number()` | `self.0 <= TYPE_NUMBER_MAX` — 判断低 6 位数值类型（F64/F32/F16/U32/I32/U40）的快速范围检查 |
| `is_f64()` | `self.0 <= TYPE_TRACED_NAN_MAX` — 利用 IEEE 754 NaN 空间的有序性做单次比较 |
| `is_non_nan_number()` | `self.0 < TYPE_NAN` — 有效 f64 总是小于 NaN 标记值 |
| `is_nan()` | 高 24 位匹配 TYPE_NAN |
| `is_nil()` | 高 24 位匹配 TYPE_NIL |
| `is_bool()` | 高 24 位匹配 TYPE_BOOL |
| `is_object()` | 高 24 位匹配 TYPE_OBJECT |
| `is_array()` | 高 24 位匹配 TYPE_ARRAY |
| `is_pod()` | 高 24 位匹配 TYPE_POD |
| `is_pod_type()` | 高 24 位匹配 TYPE_POD_TYPE |
| `is_handle()` | 高 24 位在 `TYPE_HANDLE_FIRST` 和 `TYPE_HANDLE_LAST` 之间（0x50-0x7F 范围） |
| `is_color()` | 高 24 位匹配 TYPE_COLOR |
| `is_regex()` | 高 24 位匹配 TYPE_REGEX |
| `is_string()` | 高 24 位匹配 TYPE_STRING |
| `is_string_like()` | 类型在 TYPE_STRING 到 TYPE_INLINE_STRING_END 之间（包含堆字符串和内联字符串） |
| `is_inline_string()` | 类型在 TYPE_INLINE_STRING_0 到 TYPE_INLINE_STRING_5 之间 |
| `inline_string_not_empty()` | 类型在 TYPE_INLINE_STRING_1 到 TYPE_INLINE_STRING_END 之间 |
| `is_id()` | `self.0 >= TYPE_ID` — 利用 ID 类型 0x80 是高 24 位中的最大值做快速范围检查 |
| `is_escaped_id()` | `self.0 >= TYPE_ID \| ESCAPED_ID` — 检查 ESCAPED_ID 标志位 |
| `is_opcode()` | 高 24 位匹配 TYPE_OPCODE |
| `is_index()` | `self.0 <= TYPE_NIL` — 所有数值类型和 nil 都可用于索引 |
| `is_assign_opcode()` | 提取 opcode 编号后调用 `Opcode::is_assign()` |
| `is_let_opcode()` | 检查是否为 `LET_TYPED`/`LET_DYN`/`VAR_TYPED`/`VAR_DYN` |

#### 数值类型转换

| 方法 | 实现逻辑 |
|------|----------|
| `from_f64(val)` | 如果 val 是 NaN 则返回 `NAN`；否则直接存储原始的 IEEE 754 位模式（无类型标记，因为是普通 f64） |
| `as_f64()` | 如果 `is_f64()` 返回 true，则通过 `f64::from_bits` 恢复；否则返回 None |
| `from_f64_traced_nan(val, ip)` | 如果 val 是 NaN，将其位模式与追踪 NaN 空间比较：如果在范围内则保留原值，否则用 TYPE_NAN 标记 + IP 地址构造新的追踪 NaN |
| `as_f64_traced_nan()` | 如果是追踪 NaN（`is_nan()`），从低 40 位提取 `ScriptIp`，包含 body 索引和源代码位置 |
| `from_f32(v)` | `v.to_bits() as u64 \| TYPE_F32` — 将 f32 的 32 位模式存入低 32 位 |
| `as_f32()` | 检查类型是否为 F32，然后 `f32::from_bits(self.0 as u32)` |
| `from_f16(v)` | 与 F32 相同（低 32 位存 f32 的位模式），类型标记不同 |
| `from_u32(v)` | `v as u64 \| TYPE_U32` |
| `as_u32()` | 提取低 32 位 |
| `from_i32(v)` | `v as u64 \| TYPE_I32` |
| `as_i32()` | `self.0 as i32` |
| `from_u40(v)` | `(v & 0xFF_FFFF_FFFF) \| TYPE_U40` — 40 位无符号整数 |
| `as_u40()` | 提取低 40 位 |
| `as_number()` | 依次尝试 `as_f64`、`as_u40`、`as_f32`、`as_u32`、`as_i32`、`as_f16`，返回第一个成功的 f64 值 |
| `as_index()` | 依次尝试各种数值类型和布尔，返回 `usize` 索引；失败返回 0 |

#### 对象引用类型

所有堆引用类型使用相同的低 40 位布局：
- bits 0-31: **索引**（在 GenVec 中的位置）
- bits 32-39: **代数**（当 `check_gen` 特性启用时为 8 位，每次分配递增；未启用时为零大小类型）

| 方法 | 实现逻辑 |
|------|----------|
| `from_object(ptr)` | `ptr.index as u64 \| (ptr.generation as u64) << 32 \| TYPE_OBJECT` |
| `as_object()` | 提取 index (bits 0-31) 和 generation (bits 32-39) 构造 `ScriptObject` |
| `from_array(ptr)` | 同 object，使用 TYPE_ARRAY 标记 |
| `from_pod(ptr)` | 同 object，使用 TYPE_POD 标记 |
| `from_pod_type(ptr)` | `ptr.index as u64 \| TYPE_POD_TYPE` — POD 类型没有代数 |
| `from_regex(ptr)` | 同 object，使用 TYPE_REGEX 标记 |
| `from_string(ptr)` | 同 object，使用 TYPE_STRING 标记 |

#### 句柄 (`ScriptHandle`)

句柄是外部资源的引用（如 GPU 纹理、窗口等）。类型编码在 TYPE_MASK 中：
- `TYPE_HANDLE_FIRST = 0xFFFF_FF50_0000_0000`
- 每个句柄子类型占用一个唯一的 8 位标签（0x50-0x7F，共 48 种）

```
Bits 63-40: 0xFFFF_FF + 类型偏移(0x50 + ty)   | Bits 39-32: generation | Bits 31-0: index
```

| 方法 | 实现逻辑 |
|------|----------|
| `from_handle(ptr)` | `ptr.index \| (ptr.generation << 32) \| (TYPE_HANDLE_FIRST + (ptr.ty.0 << 40))` — 句柄类型编码在高 24 位的类型字段中 |
| `is_handle()` | 高 24 位在 `TYPE_HANDLE_FIRST` 和 `TYPE_HANDLE_LAST` 之间 |
| `as_handle()` | 从高 24 位提取句柄类型：`((self.0 & TYPE_MASK) - TYPE_HANDLE_FIRST) >> 40`；从低 40 位提取 index 和 generation |

#### 布尔值

`TRUE`/`FALSE` 通过 payload 的最低位的 0/1 区分。

#### 颜色

`from_color(val)`: `val as u64 \| TYPE_COLOR` — 低 32 位存储 RGBA 值。
`as_color()`: 提取低 32 位。

#### LiveId 标识符

`from_id(val)`: `val.0 \| TYPE_ID` — LiveId 是 u64，但只使用低 46 位（因为 TYPE_ID 占用 bits 47-40 的 0x80）。
`from_escaped_id(val)`: `val.0 \| TYPE_ID \| ESCAPED_ID` — ESCAPED_ID = `0x0000_4000_0000_0000`，设置 bit 54 做逃逸标记。
`as_id()`: 提取 `self.0 & 0x0000_3fff_ffff_ffff` — 低 46 位。
逃逸 ID 用于标识符包含特殊字符时的编码。

#### 操作码

`from_opcode(op)`: `TYPE_OPCODE \| (op.0 as u64) << 32` — opcode 编号在 bits 39-32。
`from_opcode_args(op, args)`: `TYPE_OPCODE \| (op.0 << 32) \| args.0` — args 在低 32 位。
`as_opcode()`: 提取 opcode 编号和 args。
`set_opcode_args()`: 更新 args 字段而不改变 opcode。
`set_opcode_args_pop_to_me()` / `clear_opcode_args_pop_to_me()` / `has_opcode_args_pop_to_me()`: 操作 POP_TO_ME 标志位。

#### 内联字符串

内联字符串编码 **0-5 字节** 的短字符串，避免堆分配：

| 编码 | 字节数 | 布局 |
|------|--------|------|
| `TYPE_INLINE_STRING_0` | 0 字节 | `0xFFFF_FF12_0000_0000` |
| `TYPE_INLINE_STRING_1` | 1 字节 | `0xFFFF_FF13_0000_00xx` |
| `TYPE_INLINE_STRING_2` | 2 字节 | `0xFFFF_FF14_0000_xxxx` |
| `TYPE_INLINE_STRING_3` | 3 字节 | `0xFFFF_FF15_00xx_xxxx` |
| `TYPE_INLINE_STRING_4` | 4 字节 | `0xFFFF_FF16_xxxx_xxxx` |
| `TYPE_INLINE_STRING_5` | 5 字节 | `0xFFFF_FF17_xx_xxxx_xxxx` |

`from_inline_string(str)`: 根据字符串长度选择对应的 INLINE_STRING_N 类型，将字节逐字节编码到低位置。
`as_inline_string(f)`: 根据类型选择字节长度，使用 `unsafe { from_utf8_unchecked }` 重建 `&str` 并调用闭包。使用 unsafe 因为内联字符串的构建总是有效的 UTF-8。
`inline_string_not_empty()`: 检查类型是否在 INLINE_STRING_1 和 INLINE_STRING_END 之间。

#### 错误类型（19 种精简方案）

早期版本有 56 种错误类型，现在合并为 19 种：

| 构造函数 | 错误含义 |
|-----------|----------|
| `script_err_not_found(ip)` | 查找失败（属性、字段、变量、名称、索引） |
| `script_err_type_mismatch(ip)` | 类型不匹配 |
| `script_err_wrong_value(ip)` | 期望不同的值类型 |
| `script_err_out_of_bounds(ip)` | 索引越界 |
| `script_err_immutable(ip)` | 不可修改（冻结、不可赋值等） |
| `script_err_stack(ip)` | 栈溢出/下溢 |
| `script_err_invalid_args(ip)` | 参数格式错误 |
| `script_err_not_allowed(ip)` | 不允许的操作 |
| `script_err_inconsistent(ip)` | 分支间类型/名称不一致 |
| `script_err_not_impl(ip)` | 未实现 |
| `script_err_unexpected(ip)` | 通用错误 |
| `script_err_assert_fail(ip)` | 断言失败 |
| `script_err_user(ip)` | 用户生成错误 |
| `script_err_pod(ip)` | POD 相关错误 |
| `script_err_shader(ip)` | 着色器错误 |
| `script_err_unknown_type(ip)` | 未注册的类型 |
| `script_err_duplicate(ip)` | 重复键 |
| `script_err_io(ip)` | 文件系统/进程错误 |
| `script_err_limit(ip)` | 资源限制 |

每个错误使用 `err_fn!` 宏生成，将 `ScriptIp`（Body 索引 + 代码位置）编码到 payload：
```
ScriptValue(TYPE_ERR_XXX | ip.to_u40())
```

`is_err()`: 检查高 24 位是否在 ERR_FIRST 和 ERR_LAST 之间。
`as_err()`: 返回 `ValueError { ty, ip }`。

---

## 辅助类型

### `ScriptIp` (行 19-35)

表示字节码中的位置，包含 `body`（u16，body 索引）和 `index`（u32，操作码索引）。
编码/解码为 40 位值：
- `to_u40()`: `(body << 28) | index`
- `from_u40(value)`: 从 40 位值中提取 body (bits 39-28) 和 index (bits 27-0)

### `Generation` / `GENERATION_ZERO` (行 38-49)

use-after-free 检测的代数计数器。`check_gen` 特性启用时为 `u8`，否则为零大小类型 `()`。

### `ScriptPod` (行 51-84)

POD（Plain Old Data）实例索引。实现了 `From<ScriptPod> for ScriptValue`。

### `ScriptPodType` (行 86-93)

POD 类型索引。`VOID` 常量表示 void 类型。

### `ScriptObject` / `ScriptArray` (行 95-189)

堆对象和数组引用，各包含 `index` 和 `generation`。都实现了 `GenRef`  trait 以便与 `GenVec` 配合。
- `ZERO`: 空引用常量
- `new(index, generation)`: 构造函数
- `index()`: 返回堆中的索引
- `generation()`: 返回代数（`check_gen` 禁用时返回 `()`）

### `ScriptHandleType` / `ScriptHandle` (行 206-335)

句柄类型系统：`ScriptHandleType(u8)` 是 8 位类型标识符，`ScriptHandle` 包含类型、索引和代数。
- 支持最多 48 种句柄类型（0x50 到 0x7F）
- `to_redux()`: 转换为简约类型 `ScriptTypeRedux`
- `ZERO`: 空句柄常量

### `ScriptRegex` (行 381-423)

正则表达式引用，包含 index 和 generation。

### `ScriptString` (行 431-476)

堆字符串引用，包含 index 和 generation。

### `ScriptValueType` (行 593-696)

类型标签的完整定义。关键方法：

| 方法 | 逻辑 |
|------|------|
| `to_u64()` | `((self.0 as u64) << 40) \| 0xFFFF_0000_0000_0000` — 将 8 位标签放置到 bits 47-40，设置高 16 位为 1 |
| `from_u64(val)` | 提取 bits 47-40 的 8 位值，如果超过 ID 阈值(0x80)则返回 ID |
| `to_redux()` | 将详细类型映射到简约类型 `ScriptTypeRedux`：高值类型（错误→错误、句柄→句柄、ID→ID）被压缩 |

### `ScriptTypeRedux` (行 215-260)

简约类型系统，将 `ScriptValueType` 的 100+ 种标签映射到约 20 种分类。用于调度和错误消息。

---

## Trait 实现

### `From` / `Into` 转换

`ScriptValue` ↔ Rust 原生类型的转换实现了全套 `From` trait：
- `f64`, `f32`, `u32`, `i32`, `u16`, `u8`, `usize`, `bool` → 数值类型编码
- `LiveId` → ID 类型
- `Opcode` → OPCODE 类型
- `ScriptObject` / `ScriptArray` / `ScriptPod` / `ScriptPodType` / `ScriptHandle` / `ScriptRegex` / `ScriptString` → 对应引用类型
- `ScriptValue` → `ScriptObject` / `ScriptHandle` / 各种数值类型

### `fmt::Display` (行 1563-1626)

按优先级尝试解码：f64 → u40 → id → bool → 堆字符串 → 颜色 → 内联字符串 → f32 → i32 → u32 → f16 → 对象 → 数组 → 正则 → 句柄 → pod_type → pod → 错误 → nil → opcode → 回退显示十六进制。

---

## 关键设计决策

1. **NaN-boxing 而非 tagged union**: 64 位即可编码所有值类型，零堆分配开销，类型判断只需位掩码比较。
2. **40 位 payload**: 充分利用 IEEE 754 NaN 尾数空间，支持大索引（最多 2^32 个堆对象）和追踪。
3. **代数检测**: 可选的 `check_gen` 特性在开发时捕获 use-after-free，发布时零开销。
4. **内联字符串**: 0-5 字节短字符串免堆分配，覆盖大多数标识符和短文本场景。
5. **错误追踪**: 每个错误编码了 `ScriptIp`，可在运行时报错时精确定位源代码位置。
6. **f64 优先排序**: 利用 IEEE 754 位模式的有序性，`is_f64()` 只需一次比较 `self.0 <= TYPE_TRACED_NAN_MAX`。

---

## 文件信息
- **路径**: `platform/script/src/value.rs`
- **行数**: 1627
- **核心类型**: `ScriptValue(u64)`
