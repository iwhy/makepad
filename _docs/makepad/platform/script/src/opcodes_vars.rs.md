# `opcodes_vars.rs` — 变量、字段与对象操作码执行（892 行）

## 概述

本文件是 Splash VM **操作码执行器中最大的模块**，实现了：

- 对象/数组字面量构造（`BEGIN_PROTO` `END_PROTO` `BEGIN_BARE` `END_BARE` `BEGIN_ARRAY` `END_ARRAY`）
- 原型/作用域/字段/索引的继承运算符（`+:` 的多种变体）
- 字段读取（`FIELD` `FIELD_NIL` `ME_FIELD` `PROTO_FIELD`）
- `use` 导入语句
- 变量声明（`LET_TYPED` `LET_DYN` `VAR_TYPED` `VAR_DYN`）
- 属性树搜索（`SEARCH_TREE`）
- 日志输出（`LOG`）
- 上下文引用（`ME` `SCOPE`）
- 数组索引读取（`ARRAY_INDEX`）
- splat 展开（`ME_SPLAT`）
- 栈操作（`DUP` `DROP`）
- 解构赋值（`LET_DESTRUCT_ARRAY_EL` `LET_DESTRUCT_OBJECT_EL` `ARRAY_INDEX_NIL`）

---

## 方法清单与实现逻辑

### 对象/数组开始/结束

#### `handle_begin_proto()`

**原型对象字面量开始 `{ with_proto | field: val }`。**

1. 从栈弹出原型对象并解析。
2. 通过 `heap.new_with_proto_checked(proto)` 创建基于该原型的新对象（带运行时类型检查）。
3. 将 `ScriptMe::Object(me)` 压入 `mes` 栈，后续字段赋值将通过 `ASSIGN_ME` 等操作码写入此对象。
4. `goto_next()`。

#### `handle_begin_bare()`

**裸对象开始 `{ field: val }`（无原型）。**

1. 调用 `heap.new_object()` 创建空对象（无原型链）。
2. 压入 `ScriptMe::Object(me)` 到 `mes`。
3. `goto_next()`。

#### `handle_end_proto()`

**原型对象结束 `}`。**

1. 从 `mes` 栈弹出 `ScriptMe::Object(me)`。
2. 对对象调用 `heap.finalize_maybe_pod_type(me, builtins.pod)` 进行可能 Pod 类型化的最终确定。
3. 将对象引用压栈（作为字面量求值结果）。
4. `goto_next()`。

#### `handle_end_bare()`

**裸对象结束 `}`。**

1. 从 `mes` 弹出对象（无需 finalize 检查，因为无原型）。
2. 直接压栈作为字面量结果。
3. `goto_next()`。

#### `handle_begin_array()`

**数组字面量开始 `[`。**

1. 调用 `heap.new_array()` 创建新数组。
2. 将 `ScriptMe::Array(arr)` 压入 `mes`。
3. `goto_next()`。

#### `handle_end_array()`

**数组字面量结束 `]`。**

1. 从 `mes` 弹出数组。
2. 压栈作为表达式结果。
3. `goto_next()`。

---

### 继承运算符处理

这些方法实现了 `+: ` 扩展运算符的不同上下文变体，用于在对象字面量中继承并覆盖父对象的属性。

#### `handle_proto_inherit_read()`

**原型继承读取 `+: ` 的第 1 阶段。**

1. **窥视**栈顶（字段名/标识符）。
2. 获取当前 `me` 对象：
   - 查找字段在 `me` 对象原型链上的值（`proto_field_from_value`）。
   - 如果为 nil/err：清除错误，尝试通过 `proto_field_from_type_check` 检查类型系统的默认值。
3. 将继承的值压栈（供后续替换或覆盖使用）。
4. `goto_next()`。

#### `handle_proto_inherit_write()`

**原型继承写入 `+: ` 的第 2 阶段。**

1. 从栈弹出构建好的对象（替换值）和字段名。
2. 获取当前 `me` 对象，将替换后的值通过 `set_value` 写回。
3. 如果字段是字符串类，调用 `set_string_keys` 标记字符串键。
4. 压入 `NIL` 作为表达式结果 + `goto_next()`。

#### `handle_scope_inherit_read()`

**作用域继承读取（`scope +: var`）。**

1. **窥视**栈顶标识符。
2. 通过 `scope_value` 在当前作用域链中查找变量的值。
3. 如果值存在，压栈供继承使用；如果为 nil/err，清除错误标记并压入 `NIL`。
4. `goto_next()`。

#### `handle_scope_inherit_write()`

**作用域继承写入。**

1. 从栈弹出值和标识符。
2. 通过 `set_scope_value` 将值设置到作用域中（覆盖或添加）。
3. 压入 `NIL` + `goto_next()`。

#### `handle_field_inherit_read()`

**字段继承读取（`obj.field +: val`，对象字段上下文的 1 阶段）。**

1. **窥视**栈顶字段名，以及深度 1 的字段目标对象。
2. 如果目标是未转义 ID，通过 `scope_value` 解析为实际对象。
3. 在对象上查找字段值。如果为 nil/err，清除错误标记并压入 `NIL`。
4. 压栈 + `goto_next()`。

#### `handle_field_inherit_write()`

**字段继承写入（第 2 阶段）。**

1. 从栈弹出构建好的对象、字段名、目标对象。
2. 如果目标对象有效，通过 `set_value` 写入。字符串键处理同 `handle_proto_inherit_write`。
3. 压入 `NIL` + `goto_next()`。

#### `handle_index_inherit_read()`

**索引继承读取（`obj[idx] +: val` 的第 1 阶段）。**

1. 窥视索引值和目标对象/数组（支持 ID 解析）。
2. **对象路径**：通过 `value(obj, index)` 读取。
3. **数组路径**：通过 `array_index(arr, idx)` 读取。
4. 如果值为 nil/err，清除错误并压入 NIL。
5. 否则压入继承值。

#### `handle_index_inherit_write()`

**索引继承写入（第 2 阶段）。**

1. 从栈弹出构建对象、索引、目标对象/数组。
2. **对象路径**：`set_value` 写入。
3. **数组路径**：`set_array_index` 写入。
4. 压入 `NIL` + `goto_next()`。

---

### Use 处理

#### `handle_use()`

**`use obj` / `use obj::field` 作用域导入。**

1. 从栈弹出字段名和源对象。
2. **通配符路径**（`use *`）：遍历对象的所有 `map` 和 `vec` 条目，通过 `def_scope_value` 逐一定义到当前作用域。
3. **特定字段路径**（`use obj.field`）：
   - 通过 `value(obj, field)` 读取字段值。
   - 如果值不为 nil，通过 `def_scope_value` 在作用域中定义。
4. 注意：此方法在 `opcode` 分发函数中有 `return;` 语句提前退出。

---

### 字段读取

#### `handle_field()`

**字段读取 `obj.field`。（处理对象、Pod、以及原生类型的 getter 表）。**

1. 从栈弹出字段名和对象并解析。
2. **对象路径**：`heap.value(obj, field)` 在对象中查找字段。
3. **Pod 路径**：`heap.pod_read_field(pod, field, builtins.pod)` 读取 Pod 字段。
4. **原生类型 getter 路径**：通过 `value_type().to_redux()` 获取类型索引，从 `native.getters` 表中查找 getter 函数指针。通过 unsafe 指针调用 getter 获取字段值。
5. 值压栈 + `goto_next()`。

#### `handle_field_nil()`

**安全字段访问 `obj?.field`。**

1. 弹出字段名和对象。
2. **对象路径**：`heap.value(obj, field)` 读取字段值并压栈。
3. **非对象路径**：直接压入 `NIL`（不会报错）。
4. `goto_next()`。

#### `handle_me_field()`

**当前上下文字段读取 `me.field`（在对象/函数字面量内部使用）。**

1. 弹出字段名。
2. 获取当前 `me` 上下文：
   - `ScriptMe::Object(obj)` / `ScriptMe::Call { args }`：`heap.value` 读取字段。
   - `ScriptMe::Pod { pod }`：`pod_read_field` 读取。
   - `ScriptMe::Array`：生成错误。
3. 值压栈 + `goto_next()`。

#### `handle_proto_field()`

**原型链字段读取 `proto.field`。**

1. 弹出字段名和对象。
2. 通过 `proto_field_from_value` 在对象的原型链上查找字段。
3. 如果值为 nil/err：清除错误标记，尝试 `proto_field_from_type_check` 从类型系统的默认值中查找。
4. 如果第二次查找也失败且字段不是标识符，生成 `script_err_not_found!` 错误。
5. 值压栈 + `goto_next()`。

#### `handle_pop_to_me()`

**显式 `pop_to_me` 操作码。**

1. 调用 `self.pop_to_me()`（实现在 `opcodes_loops.rs`）。
2. `goto_next()`。

#### `handle_me_splat()`

**Splat 展开操作符 `..source`（将集合展开到当前构造上下文中）。**

1. 从栈弹出源集合。
2. 仅在 `call_has_me()` 时执行。
3. 根据 `me` 类型分发：
   - **`ScriptMe::Object(obj)`**：
     - 源是对象 → `heap.merge_object(obj, source_obj)` 合并所有属性。
     - 源是数组 → 遍历数组，通过 `vec_push(obj, NIL, value)` 将每个元素追加到对象的 vec 中。
   - **`ScriptMe::Array(arr)`**：
     - 源是数组 → `heap.merge_array(arr, source_arr)` 合并所有元素。
     - 源是对象 → `heap.array_push_vec(arr, source_obj)` 将对象作为命名参数向量追加。
   - **`ScriptMe::Call { args }`**：
     - 源是对象 → 遍历 `vec_len`，通过 `unnamed_fn_arg(args, value)` 追加位置参数。
     - 源是数组 → 遍历数组长度，通过 `unnamed_fn_arg` 追加每个元素。
   - **`ScriptMe::Pod`**：生成 `script_err_not_impl!` 错误（Pod 不支持 splat）。
4. `goto_next()`。

---

### 数组索引读取

#### `handle_array_index()`

**数组/对象索引读取 `obj[idx]`。**

1. 从栈弹出索引和对象。
2. **对象路径**：`heap.value(obj, index)` 用索引作为键查找。
3. **数组路径**：`heap.array_index(arr, index.as_index())` 用整数索引读取。
4. **Pod 路径**：`heap.pod_array_index(pod, index, builtins.pod)`。
5. 如果三者都不是，生成 `script_err_wrong_value!` 错误。
6. 值压栈 + `goto_next()`。

---

### 变量声明

#### `handle_let_dyn(opargs)`

**`let id = value` 动态类型声明。**

1. 从 `opargs` 判断是否有初始值值（`is_nil()` 表示无，使用 `NIL`）。
2. 从栈弹出标识符。
3. 通过 `def_scope_value` 在当前作用域定义变量（初始化）。
4. `goto_next()`。

#### `handle_let_typed(opargs)`

**`let id: Type = value` 类型标注声明。**

1. 弹出值（或 nil）、类型值、标识符。
2. 当前忽略类型值（`_ty`，仅编译期用于类型检查）。
3. 通过 `def_scope_value` 定义变量。
4. `goto_next()`。

#### `handle_var_dyn(opargs)`

**`var id = value` 动态类型变量声明（可变）。**

与 `handle_let_dyn` 完全相同——在当前作用域定义变量。区别仅在语法层面（`let` 为不可变绑定，`var` 为可变；但 VM 层面不强制）。

#### `handle_var_typed(opargs)`

**`var id: Type = value` 类型标注可变声明。**

与 `handle_let_typed` 完全相同。

---

### 搜索树

#### `handle_search_tree()`

**属性树搜索 `$`。**

1. 直接 `goto_next()`。（搜索树逻辑在编译期由编译器处理推导，运行时不执行任何操作。）

---

### 日志

#### `handle_log()`

**`log(value)` 日志输出。**

1. **窥视**栈顶值。
2. 调用 `self.log(value)` 输出日志。
3. `goto_next()`。

---

### Me/Scope 上下文

#### `handle_me()`

**获取当前构造上下文 `me` 关键字。**

1. 如果 `call_has_me()`：根据当前 `me` 类型获取引用：
   - `Object(obj)` / `Call { args }` / `Array(arr)` / `Pod { pod }` → 对象引用。
2. 如果没有 me：返回 `NIL`。
3. 值压栈 + `goto_next()`。

#### `handle_scope()`

**获取当前作用域对象 `scope` 关键字。**

1. 调用 `scopes.last()` 获取作用域链顶层的 scope 对象。
2. 如果为空，调用 `self.bail()` 报告错误。
3. 值压栈 + `goto_next()`。

---

### 日志实现

#### `log(value)`

**日志输出的完整实现。**

1. 获取当前 IP 对应的源码位置（`ip_to_loc`）。
2. 根据值的类型分情况输出：
   - **`NIL`** → 输出 `"nil"`。
   - **错误值**（`value.as_err()`）→ 在错误队列中查找对应错误，格式化输出错误消息、原文件、行号、位置。
   - **NaN 追踪值**（`value.as_f64_traced_nan()`）→ 输出值和 NaN 来源位置。
   - **其他值** → 通过 `heap.to_debug_string()` 生成可读的调试字符串，输出值类型和内容。

---

### 栈操作

#### `handle_dup()`

**复制栈顶值 `dup`。**

1. 窥视栈顶值并解析。
2. `push_stack_unchecked` 压入复制的值。
3. `goto_next()`。

#### `handle_drop()`

**丢弃栈顶值 `drop`。**

1. `pop_stack_value()` 弹出并丢弃。
2. `goto_next()`。

---

### 解构赋值

#### `handle_array_index_nil()`

**安全索引访问 `?[]`（nil 安全、错误安全）。**

1. 从栈弹出索引和源。
2. **数组路径**：`heap.array_index(arr, idx)` → 如果结果是错误，清除错误并返回 `NIL`。
3. **对象路径**：`heap.value(obj, index)` → 类似地错误转为 `NIL`。
4. **其他路径**：直接返回 `NIL`。
5. 值压栈 + `goto_next()`。

#### `handle_let_destruct_array_el(opargs)`

**数组解构元素绑定 `[a, b] = source`。**

1. `opargs.to_u32()` 获取元素索引。
2. 从栈弹出标识符和源（`[source, id]` 栈布局）。
3. **数组路径**：`heap.array_index(arr, index)` → 如果错误，清除错误用 `NIL`。
4. **对象路径**：`heap.value(obj, u32_to_value(index))` → 如果错误，清除错误用 `NIL`。
5. **其他路径**：`NIL`。
6. 如果标识符有效，通过 `def_scope_value` 绑定到当前作用域。
7. **源重新压栈**：将 `source` 推回栈顶，供后续元素继续解构。
8. `goto_next()`。

#### `handle_let_destruct_object_el()`

**对象解构元素绑定 `{ key } = source`。**

1. 从栈弹出标识符和源（`[source, id]`）。
2. 如果源是对象：`heap.value(obj, id)` 用标识符本身作为键查找值。错误转为 `NIL`。
3. 其他路径：`NIL`。
4. 如果标识符有效，绑定到作用域。
5. **源重新压栈**供后续元素解构。
6. `goto_next()`。
