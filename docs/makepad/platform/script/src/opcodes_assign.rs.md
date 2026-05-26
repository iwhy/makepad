# `opcodes_assign.rs` — 赋值操作码执行（767 行）

## 概述

本文件实现了 Splash VM 中所有**赋值操作码**的执行逻辑，包括四大家族：

1. **作用域赋值**（`ASSIGN_*`）— 对当前作用域中的标识符赋值
2. **字段赋值**（`ASSIGN_FIELD_*`）— 对对象的命名字段赋值
3. **索引赋值**（`ASSIGN_INDEX_*`）— 对对象/数组的索引赋值
4. **me 赋值**（`ASSIGN_ME_*`）— 对当前构造上下文的对象赋值

以及它们的复合变体（`+=` `-=` `*=` `/=` `%=` 等）和 nil 安全变体（`?=`）。

---

## 方法清单与实现逻辑

### 作用域赋值

#### `handle_assign()`

**普通赋值 `=`（scope 变量赋值）。**

1. 从栈弹出右值（`value`）和标识符（`id`）。
2. 如果 `id` 是有效的 `LiveId`：
   - 调用 `cur().set_scope_value(&mut heap, id, value)` 将在当前作用域设置变量值。
   - 将设置后的值压回栈（赋值表达式的值是右值）。
3. 如果不是标识符，通过 `script_err_immutable!` 宏生成错误值压栈。
4. 调用 `goto_next()`。

#### `handle_assign_add()`

**复合赋值 `+=`（scope 变量，支持字符串拼接）。**

1. 从栈弹出右值和标识符。
2. 通过 `scope_value()` 读取变量的旧值。
3. 字符串快速路径：如果旧值或新值是字符串类，使用 `new_string_with` 拼接并设置。
4. 否则 `cast_to_f64()` 后执行 `fa + fb`，设置结果。
5. 如果标识符无效，生成错误值。
6. 调用 `goto_next()`。

#### `handle_assign_ifnil(opargs)`

**nil 合并赋值 `?=`（仅当 scope 变量为 nil 时才执行赋值）。**

1. **窥视**（不弹出）栈顶标识符。
2. 如果标识符在 scope 中已存在且值不是 nil/err：
   - 弹出标识符，压入 nil 作为表达式结果。
   - 跳转到 `opargs.to_u32()` 指定的偏移，跳过 RHS 和实际赋值操作。
3. 如果标识符不存在或值为 nil：
   - 保留标识符在栈上，继续执行 RHS 求值。后续由 `ASSIGN` 操作码完成赋值。
4. 如果标识符无效，生成错误值。
5. 调用 `goto_next()`（或通过 `goto_rel` 跳转）。

### 泛型作用域辅助方法

#### `handle_f64_scope_assign_op(f)` — 浮点复合赋值（`-=` `*=` `/=` `%=`）

1. 弹出右值，弹出标识符。
2. 读取 scope 中的旧值。
3. 如果旧值不是错误，`cast_to_f64` 转换后执行 `f(fa, fb)`。
4. 设置新值并压栈。
5. 如果不是标识符，生成错误。

#### `handle_fu64_scope_assign_op(f)` — 整数位运算复合赋值（`&=` `|=` `^=` `<<=` `>>=`）

1. 与浮点版本流程相同，但数值通过 `cast_to_f64() as u64` 转为无符号整数运算。
2. 结果再转回 `f64` 存储。

---

### 字段赋值

#### `handle_assign_field()`

**字段赋值 `obj.field = val`。**

1. 从栈弹出右值、字段名、对象（三元素栈布局）。
2. 如果对象是 `ScriptObject`：通过 `heap.set_value(obj, field, value)` 设置字段值。
3. 如果不是对象，生成 `script_err_wrong_value!` 错误。
4. 赋值后的值压栈 + `goto_next()`。

#### `handle_assign_field_add()`

**字段复合赋值 `obj.field += val`（支持字符串拼接）。**

1. 弹出右值、字段名、对象。
2. 读取字段旧值：`heap.value(obj, field)`。
3. **字符串路径**：若任一操作数字符串类，拼接后设置。
4. **数值路径**：`cast_to_f64` 后运算。
5. 如果不是对象，生成 `script_err_immutable!` 错误。

#### `handle_assign_field_ifnil()`

**字段 nil 合并赋值 `obj.field ?= val`。**

1. 弹出右值、字段名、对象。
2. 读旧值：若旧值是 nil 或 err → 设置新值；否则压入 nil（不修改）。
3. 若目标不是对象，生成 `script_err_wrong_value!`。

### 泛型字段辅助方法

#### `handle_f64_field_assign_op(f)`

字段浮点复合赋值通用方法（`-=` `*=` `/=` `%=`）。与 `handle_assign_field_add` 类似，但直接调用闭包 `f` 执行运算。

#### `handle_fu64_field_assign_op(f)`

字段位运算复合赋值通用方法（`&=` `|=` `^=` `<<=` `>>=`）。数值转为 `u64` 再运算。

---

### 索引赋值

#### `handle_assign_index()`

**索引赋值 `obj[idx] = val`。**

1. 从栈弹出右值、索引、对象。
2. 特殊处理：如果索引是转义 ID（变量引用），通过 `ScriptValue::from_id()` 解转义。
3. **对象路径**：调用 `heap.set_value(obj, index, value)`。
4. **数组路径**：调用 `heap.set_array_index(arr, index.as_index(), value)`。
5. 如果既不是对象也不是数组，生成错误值。
6. 赋值结果压栈 + `goto_next()`。

#### `handle_assign_index_add()`

**索引复合赋值 `obj[idx] += val`（支持字符串拼接）。**

1. 弹出右值、索引、对象。
2. **对象路径**：读旧值 → 若字符串则拼接 → 数值则 `cast_to_f64` 相加 → `set_value`。
3. **数组路径**：读旧值 → 类似判断 → `set_array_index`。
4. 若都不是，生成错误。

#### `handle_assign_index_ifnil()`

**索引 nil 合并赋值 `obj[idx] ?= val`。**

1. 弹出右值、索引、对象。
2. **对象路径**：读旧值 → 若 nil/err 则设置，否则返回 nil。
3. **数组路径**：类似逻辑。
4. 若都不是，生成错误。

### 泛型索引辅助方法

#### `handle_f64_index_assign_op(f)` / `handle_fu64_index_assign_op(f)`

分别为浮点和整数位运算的索引复合赋值通用方法。支持对象和数组两条路径，逻辑与字段版本对称。

---

### me 赋值

这些操作码用于对象/数组/函数字面量构造过程中，将命名字段赋值到当前构造上下文（`me`）。

#### `handle_assign_me()`

**命名属性赋值 `name: value`（对象字面量内部）。**

1. 从栈弹出右值和字段名。
2. 检查 `call_has_me()`，如果有 `me`：
   - 调用 `mes.last()` 获取当前 `me` 上下文：
     - `ScriptMe::Call { args }` → `heap.named_fn_arg(args, field, value)` 设置函数调用命名参数
     - `ScriptMe::Object(obj)` → 若字段是字符串则 `set_string_keys`，然后 `heap.set_value(obj, field, value)`
     - `ScriptMe::Pod { pod }` → `heap.set_pod_field(pod, field, value)`
     - `ScriptMe::Array` → 生成错误（数组字面量不支持命名属性）
3. `goto_next()`。

#### `handle_assign_me_vec()`

**vec 命名属性赋值 `name := value`（push 语义）。**

与 `handle_assign_me()` 逻辑相同，唯一区别是对 `ScriptMe::Object(obj)` 路径调用 `heap.set_value_vec()` 而非 `set_value()`。`set_value_vec` 将属性作为有序列表项追加到对象的 `vec` 中，而非设置到 `map`。

#### `handle_assign_me_before_after(opcode)`

**插入到命名属性之前 `:< ` 或之后 `:>`。**

1. 弹出右值和字段名。
2. 获取当前 `me`：
   - `ScriptMe::Object(obj)` → 调用 `heap.vec_insert_value_at(obj, field, value, is_before)`。
   - `ScriptMe::Call` / `ScriptMe::Pod` / `ScriptMe::Array` → 生成错误。
3. 将插入结果压栈 + `goto_next()`。

#### `handle_assign_me_begin()`

**插入到对象 vec 起始 `:<<`。**

1. 弹出右值和字段名。
2. 获取当前 `me`：
   - `ScriptMe::Object(obj)` → 调用 `heap.vec_insert_value_begin(obj, field, value)`。
   - 其他上下文 → 生成错误。
3. 结果压栈 + `goto_next()`。
