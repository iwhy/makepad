# `opcodes_control.rs` — 控制流操作码执行（347 行）

## 概述

本文件实现了 Splash VM 中**控制流**相关的操作码执行逻辑，包括：

- `if`/`else` 条件分支
- `return` 函数返回
- `?` 错误传播
- `for` 循环的初始化（循环体迭代见 `opcodes_loops.rs`）
- `loop` 无限循环
- `break` / `continue` / `breakifnot` 循环控制
- `range` 对象创建
- `is` 类型检查操作符
- `try`/`ok` 异常处理
- 短路求值（`&&` `||` `|?`）

---

## 方法清单与实现逻辑

### IF 处理

#### `handle_if_test(opargs)`

**`if` 条件测试。**

1. 从栈弹出条件值并解析。
2. 通过 `heap.cast_to_bool(test)` 将值转为布尔。
3. **分支逻辑**：
   - 如果为 `true`：执行 `trap.goto_next()` 进入 `if` 分支。
   - 如果为 `false`：
     - 如果 `opargs.is_need_nil()`：压入 `NIL` 作为 if 表达式的结果值。
     - 执行 `trap.goto_rel(opargs.to_u32())` 跳转到 `else` 分支或跳过整个 `if` 块。
4. 注意：`goto_rel` 的偏移由编译器计算并编码在参数中。

#### `handle_if_else(opargs)`

**`else` 分支跳转（跳过 `else` 块进入 if 后续代码）。**

1. 直接执行 `trap.goto_rel(opargs.to_u32())` 跳转到 if-else 结构之后的代码。
2. 无栈操作，仅修改指令指针。

---

### RETURN 处理

#### `handle_return(opargs)`

**函数返回操作。**

1. 根据 `opargs.is_nil()` 判断是否有返回值：
   - is_nil → 返回 `NIL`（无返回值函数）。
   - 否则从栈弹出返回值并解析。
2. 从 `calls` 栈弹出最近的调用帧（`CallFrame`）：
   - 调用 `truncate_bases(call.bases)` 清理调用后作用域和基础帧。
3. **有返回位置**（普通函数调用 `return_ip.is_some()`）：
   - 设置 `trap.ip = ret` 指向调用点之后的下一条指令。
   - 将返回值压栈。
   - 如果 `call.args.is_pop_to_me()`，调用 `self.pop_to_me()` 将返回值注入调用方的 `me` 上下文。
4. **无返回位置**（顶层代码、主脚本）：
   - 设置 `trap.on = Some(ScriptTrapOn::Return(value))` 触发脚本完成信号。

#### `handle_return_if_err(_opargs) -> bool`

**错误传播操作符 `?`。**

1. **窥视**栈顶值（不弹出），检查是否为错误值。
2. **错误路径**（`value.is_err()`）：
   - 从 `calls` 栈弹出调用帧。
   - `truncate_bases` 清理。
   - 如果 `return_ip` 存在：跳转到返回位置，错误值压栈，可能调用 `pop_to_me`。
   - 如果不存在：触发 `ScriptTrapOn::Return`。
   - 返回 `true`（指示 caller 需要处理 `pop_to_me`）。
3. **正常路径**（非错误）：
   - `goto_next()` 继续执行后续代码。
   - 返回 `false`。

---

### 循环初始化

#### `handle_for_1(opargs)`

**`for v in source` — 单变量 for 循环。**

1. 从栈弹出集合源（`source`）和值变量 ID（`value_id`）。
2. 调用 `begin_for_loop(opargs.to_u32(), source, value_id, None, None)`。
3. 无 `index_id` 或 `key_id`。

#### `handle_for_2(opargs)`

**`for k, v in source` — 键值变量 for 循环。**

1. 弹出 source、value_id、first_id（键/索引变量 ID）。
2. 传递 `key_id = Some(first_id)` 给 `begin_for_loop`。
3. 对对象：k 为 key；对数组/range：k 为 index。

#### `handle_for_3(opargs)`

**`for i, k, v in source` — 索引+键+值 for 循环（仅限对象）。**

1. 弹出 source、value_id、key_id、index_id（四个值）。
2. 传递 `index_id` 和 `key_id` 给 `begin_for_loop`。

#### `handle_loop(opargs)`

**无限循环 `loop { ... }`。**

1. 调用 `begin_loop(opargs.to_u32())`。
2. 与 for 循环不同，loop 没有 source、value_id 等，仅记录跳转偏移用于退出。

---

### 循环控制

#### `handle_for_end()`

**for 循环迭代步进（在每次循环体执行完毕后执行）。**

1. 调用 `end_for_loop()`（实现在 `opcodes_loops.rs`）。
2. 该方法处理迭代逻辑：推进索引、检查边界、创建新作用域、设置下一轮循环变量值或跳出。

#### `handle_break()`

**`break` 中断循环。**

1. 调用 `break_for_loop()`（实现在 `opcodes_loops.rs`）。
2. 该方法弹出一个 `LoopFrame`，回退作用域和栈基础，跳转到循环后的代码。

#### `handle_breakifnot()`

**`break if not` — 条件循环中断。**

1. 从栈弹出条件值并解析。
2. 如果条件为假（`!cast_to_bool(value)`）：调用 `break_for_loop()`。
3. 如果条件为真：`goto_next()` 继续执行。

#### `handle_continue()`

**`continue` 继续下一轮循环。**

1. 为 for 循环：调用 `end_for_loop()` 推进迭代器到下一元素（与 `handle_for_end` 相同效果）。
2. 为 loop 循环：直接跳转到循环起始位置（由 `end_for_loop` 的 plain 路径处理）。

---

### Range 处理

#### `handle_range()`

**`start..end` 范围对象创建。**

1. 从栈弹出结束值和起始值并解析。
2. **类型检查**：
   - 起始值不是数字 → 生成 `script_err_type_mismatch!` 错误，提示可能缺少逗号或 splat 运算符。
   - 结束值不是数字 → 生成类型不匹配错误。
3. 基于内置的 `range` 原型创建新对象：`heap.new_with_proto(builtins.range.into())`。
4. 通过 `set_value_def` 分别设置 `start` 和 `end` 字段。
5. 将 range 对象压栈 + `goto_next()`。

---

### Is 处理

#### `handle_is()`

**`value is type` 类型检查操作符。**

1. 从栈弹出 RHS（类型标识符或值）和 LHS（待检查的值）。
2. **标识符路径**（RHS 是 ID）：
   - 获取 LHS 的 `value_type()`，与常用类型名比较：
     - `number`：匹配 `F64 | F32 | F16 | U32 | I32 | U40` 六种数值类型。
     - `nan`：匹配 `NAN` 或 `number`。
     - `bool` / `nil` / `color` / `array` / `regex` / `string` / `id`：简单类型匹配。
     - `object`：匹配 OBJECT 类型；或者如果 RHS 是一个 scope 中的命名对象，检查 LHS 的原型链是否包含该对象（通过 `heap.has_proto`）。
     - 其他类型 → `false`。
3. **nil 值路径**（RHS 是 nil 字面量）：检查 `lhs.is_nil()`。
4. **原型链路径**（RHS 是对象）：通过 `heap.has_proto(lhs_obj, rhs)` 沿原型链检查。
5. 其他情况 → `false`。
6. 结果以 `bool` 形式压栈 + `goto_next()`。

---

### 短路求值处理

#### `handle_logic_or_test(opargs)`

**`||` 短路求值测试。**

1. **窥视**栈顶值。
2. 转为布尔：如果为真（truthy）：
   - 保留值在栈上，通过 `goto_rel` 跳过第二个操作数。
3. 如果为假（falsy）：
   - 弹出当前值，通过 `goto_next()` 进入第二个操作数的求值。

#### `handle_logic_and_test(opargs)`

**`&&` 短路求值测试。**

1. 窥视栈顶值，转为布尔。
2. 如果为假（falsy）：保留值在栈上，`goto_rel` 跳过第二个操作数。
3. 如果为真（truthy）：弹出当前值，`goto_next()` 进入第二个操作数。

#### `handle_nil_or_test(opargs)`

**`|?` nil 合并短路求值测试。**

1. 窥视栈顶值。
2. 如果值不为 nil：保留值，`goto_rel` 跳过第二个操作数。
3. 如果值为 nil：弹出 nil，`goto_next()` 进入第二个操作数。

---

### Try / OK 异常处理

#### `handle_ok_test(opargs)`

**`ok { ... }` 块开始。**

1. 获取当前 IP 和 `bases`。
2. 将 `TryFrame { push_nil: true, start_ip, jump, bases }` 压入 `tries` 栈。`push_nil: true` 表示该帧在出错时自动压入 nil。
3. 跳转偏移为 `opargs.to_u32() + 1`。
4. `goto_next()` 进入 ok 块内容。

#### `handle_ok_end()`

**`ok` 块结束。**

1. 从 `tries` 栈弹出最近的一个 `TryFrame`（清理异常处理帧）。
2. `goto_next()`。

#### `handle_try_test(opargs)`

**`try { ... }` 块开始。**

1. 获取当前 IP 和 `bases`。
2. 将 `TryFrame { push_nil: false, start_ip, jump, bases }` 压入 `tries` 栈。`push_nil: false` 表示保留错误值。
3. 跳转偏移为 `opargs.to_u32() + 1`。
4. `goto_next()` 进入 try 块内容。

#### `handle_try_err(opargs)`

**`try` 的 catch 分支入口。**

1. 从 `tries` 栈弹出 `TryFrame`。
2. 如果 `tries` 空：调用 `self.bail()` 报告错误。
3. 通过 `goto_rel(opargs.to_u32() + 1)` 跳转到 catch 分支的起始位置。
4. 该偏移由编译器编码，指向 `catch` 子句的入口。

#### `handle_try_ok(opargs)`

**`try` 的成功分支跳转（跳过 catch 分支）。**

1. 直接调用 `trap.goto_rel(opargs.to_u32())` 跳过 catch 块。
2. 无栈操作，仅修改指令指针。
