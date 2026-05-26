# `opcodes_ops.rs` — 算术、比较与逻辑操作码执行

## 概述

本文件实现了 Splash VM 中所有**算术运算、比较运算、字符串拼接和逻辑短路测试**操作码的执行逻辑。所有方法作为 `ScriptVm` 的 `pub(crate)` 方法实现。

---

## 方法清单与实现逻辑

### `handle_not()`

**一元逻辑/位运算非操作。**

1. 从栈弹出操作数并解析为 `ScriptValue`。
2. 尝试调用 `as_f64()` 将值转为 `f64`：
   - 如果成功，将值按 `u64` 位取反，再转回 `f64` 压栈（实现按位非 `~` 的语义）。
   - 如果失败，调用 `heap.cast_to_bool()` 将值转为布尔，取逻辑非后压栈（实现 `!` 的语义）。
3. 调用 `goto_next()` 推进指令指针。

### `handle_neg()`

**一元数值取负操作。**

1. 从栈弹出操作数并解析。
2. 优先尝试 `as_number()` 获取数值（`f64`）：
   - 成功则计算 `-f` 直接压栈返回。
3. 否则使用 `NumericValue::from_script_value_heap()` 将值转为通用数值表示（支持 `f32`/`f16`/`u40` 等多种数值类型）：
   - 构造 `F64(-1.0)` 作为因子。
   - 调用 `zip_f32(numeric_neg_one, |a, b| a * b)` 通过乘法实现取负。
   - 将结果再转回 `ScriptValue` 压栈。
4. 调用 `goto_next()`。

### `handle_add(opargs: OpcodeArgs)`

**加法（支持数字和字符串拼接）。**

1. 根据参数类型决定右操作数来源：
   - 若 `opargs.is_u32()`，右操作数为 `opargs.to_u32()` 的立即数。
   - 否则从栈弹出右操作数并解析。
2. 从栈弹出左操作数并解析。
3. **字符串拼接快速路径**：如果左或右操作数是字符串类值（`is_string_like()`），则：
   - 调用 `heap.new_string_with()` 分配新字符串。
   - 闭包内分别将两个值 `cast_to_string()` 写入字符串缓冲区。
   - 压入新字符串指针。
4. **数字快速路径**：如果两个操作数都可转为 `as_number()`，则：
   - 计算 `fa + fb`，使用 `from_f64_traced_nan()` 做 NaN 追踪，压栈。
5. **通用数值路径**：使用 `NumericValue` 进行数值运算，支持混合数值类型。
6. 调用 `goto_next()`。

### `handle_concat()`

**字符串拼接操作。**

1. 从栈弹出两个操作数并解析。
2. 调用 `heap.new_string_with()` 分配新字符串：
   - 闭包内先 `cast_to_string(op1)` 再 `cast_to_string(op2)`。
3. 将新字符串指针压栈。
4. 调用 `goto_next()`。

### `handle_eq()`

**深度相等比较 `==`。**

1. 从栈依次弹出右操作数和左操作数并解析。
2. 调用 `heap.deep_eq(a, b)` 进行深度比较（递归比较对象/数组的全部字段/元素）。
3. 将 `bool` 结果压栈。
4. 调用 `goto_next()`。

### `handle_neq()`

**深度不等比较 `!=`。**

1. 与 `handle_eq()` 相同方式弹出左右操作数。
2. 调用 `heap.deep_eq(a, b)` 后取逻辑非 `!`。
3. 将结果压栈。
4. 调用 `goto_next()`。

### `handle_shallow_eq()`

**浅相等比较 `===`。**

1. 弹出栈顶两个操作数并解析。
2. 使用 Rust 的 `ScriptValue::==` 运算符（比较值的类型和内容，不递归）。
3. 结果 `push_stack_value()` 压栈（注意此处使用 `push_stack_value` 而非 `push_stack_unchecked`）。
4. 调用 `goto_next()`。

### `handle_shallow_neq()`

**浅不等比较 `!==`。**

1. 实现与 `handle_shallow_eq()` 完全相同，只是结果取反。
2. 使用 `push_stack_unchecked` 压栈。
3. 调用 `goto_next()`。

### `handle_f64_op(args, f)` — 泛型 64 位浮点运算

**用于实现 `MUL` 之外的其他算术运算（已单独实现）的泛型模板。**

1. 获取当前 IP（用于 NaN 追踪）。
2. 右操作数：若 `args.is_u32()` 则用立即数；否则从栈弹出并 `cast_to_f64()`。
3. 左操作数：从栈弹出并 `cast_to_f64()`。
4. 调用 `f(fa, fb)` 执行闭包运算（如 `a % b`）。
5. 结果 `from_f64_traced_nan` 压栈 + `goto_next()`。

### `handle_fu64_op(args, f)` — 泛型 64 位无符号整数运算

**用于实现位运算（`<<` `>>` `&` `|` `^`）。**

1. 右操作数：若 `args.is_u32()` 用立即数；否则从栈弹出，`cast_to_f64()` 后转 `u64`。
2. 左操作数：从栈弹出，`cast_to_f64()` 后转 `u64`。
3. 调用 `f(ua, ub)` 执行位运算。
4. 结果转 `f64` 后 `from_f64_traced_nan` 压栈 + `goto_next()`。

### `handle_f64_cmp_op(args, f)` — 泛型浮点比较运算

**用于实现 `<` `>` `<=` `>=`。**

1. 右操作数：若 `args.is_u32()` 用立即数；否则从栈弹出并 `cast_to_f64()`。
2. 左操作数：从栈弹出并 `cast_to_f64()`。
3. 调用 `f(fa, fb)`（如 `|a, b| a < b`），返回 `bool`。
4. 使用 `from_bool()` 将结果压栈 + `goto_next()`。

### `handle_mul(args)`

**乘法（优先走数字快速路径）。**

1. 与 `handle_add` 类似的参数获取逻辑。
2. **数字快速路径**：若两操作数都可 `as_number()`，直接 `fa * fb` 压栈。
3. **通用数值路径**：使用 `NumericValue::multiply()` 方法，支持多种数值类型的混合乘法。
4. 调用 `goto_next()`。

### `handle_div(args)`

**除法（优先走数字快速路径，除零安全）。**

1. 与 `handle_mul` 相同的参数获取逻辑。
2. **数字快速路径**：直接 `fa / fb` 压栈。
3. **通用数值路径**：使用 `zip_f32(nb, |x, y| if y != 0.0 { x / y } else { 0.0 })` 实现除零保护。
4. 调用 `goto_next()`。

### `handle_sub(args)`

**减法（优先走数字快速路径）。**

1. 与 `handle_mul` 相同的参数获取逻辑。
2. **数字快速路径**：直接 `fa - fb` 压栈。
3. **通用数值路径**：使用 `zip_f32(nb, |x, y| x - y)`。
4. 调用 `goto_next()`。

---

## 关键技术点

- **NaN 追踪**：使用 `from_f64_traced_nan(result, ip)` 在产生 `NaN` 时记录当前指令位置，便于调试追踪 `NaN` 的来源。
- **数值类型混合运算**：通过 `NumericValue` 体系支持 `f64`/`f32`/`f16`/`u32`/`i32`/`u40` 六种数值类型之间的无缝混合运算。
- **立即数优化**：当右操作数是编译时可确定的小整数时，编码在 `OpcodeArgs` 中直接使用，无需栈操作。
- **字符串拼接优化**：在 `handle_add` 中自动检测字符串类型，走字符串拼接路径而非报错。
