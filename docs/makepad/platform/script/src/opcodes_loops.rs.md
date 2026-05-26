# `opcodes_loops.rs` — 循环迭代辅助函数（469 行）

## 概述

本文件实现了 Splash VM 中 **for 循环和无限循环的内核迭代逻辑**。包含循环帧的创建、迭代推进、中断、继续、以及 `pop_to_me`（将值注入构造上下文）的核心实现。

---

## 方法清单与实现逻辑

### `begin_for_loop_inner(jump, source, value_id, index_id, key_id, first_value, first_index, first_key)`

**for 循环内部初始化（所有变体的统一入口）。**

1. 调用 `trap.goto_next()` 推进到循环体第一条指令。
2. 记录当前 `bases`（基础帧位置）和 `start_ip`（循环起始指令位置，用于跳回）。
3. 构造 `LoopFrame` 并压入 `loops` 栈：
   - `bases`：记录当前作用域/基础帧快照。
   - `start_ip`：循环体的起始 IP。
   - `values`：包含 `value_id`、`key_id`、`index_id`、`source`（集合源引用）、`index`（当前迭代索引）的 `LoopValues` 结构。
   - `jump`：循环结束后的跳转偏移（用于在循环完成时跳出）。
4. 创建新作用域对象：
   - 调用 `scopes.last()` 获取父作用域。
   - 通过 `heap.new_with_proto(parent_scope)` 创建子作用域（使用原型链继承父作用域）。
   - 调用 `clear_object_deep` 清空。
   - 通过 `set_value_def` 将第一次迭代的值设置到新作用域中：
     - `value_id`：值变量（必需）。
     - `key_id`：如果指定，对于数组传入 `first_index`（数组无键，索引作为键），对于对象传入 `first_key`。
     - `index_id`：如果指定，传入 `first_index`。
5. 新作用域入栈。

### `begin_loop(jump)`

**无限循环初始化。**

1. `goto_next()` 推进到循环体第一条指令。
2. 记录 `bases` 和 `start_ip`。
3. 压入 `LoopFrame { values: None, ... }`（无迭代变量）。
4. 创建子作用域（与 for 循环相同，但不设置任何迭代变量）。
5. 新作用域入栈。

### `begin_for_loop(jump, source, value_id, index_id, key_id)`

**for 循环的顶级入口（在 `handle_for_1/2/3` 中调用）。**

1. 构造初始值 `v0 = ScriptValue::from_f64(0.0)`。
2. **纯数值迭代**（`for v in 5`，数字作为集合源）：
   - 如果 source 是数字且 `>= 1.0`：调用 `begin_for_loop_inner` 传入初始值 `0.0`。
   - 数字 `n` 表示迭代 `0..n`。
3. **Range 对象迭代**（`for v in 1..10`）：
   - 调用 `heap.has_proto(obj, builtins.range)` 检查原型是否为 range。
   - 读取 `start` 和 `end` 字段。
   - 如果 `|start - end| >= 1.0`；调用 `begin_for_loop_inner` 传入起始值和索引。
4. **对象迭代**（`for k, v in obj`）：
   - 通过 `heap.iter_len(obj)` 获取对象可迭代长度。
   - 如果长度 > 0，通过 `heap.iter_key_value(obj, 0)` 获取第一对键值，传入 `begin_for_loop_inner`。
5. **数组迭代**（`for v in arr`）：
   - 对于 `for i k v in arr`（index_id + key_id 都指定）：生成错误（不支持）。
   - 如果 `array_len > 0`：获取 `array_index(arr, 0)` 作为第一个值，传入 `begin_for_loop_inner`。
6. 如果集合为空或不支持的类型：通过 `trap.goto_rel(jump)` 跳过整个循环。

### `end_for_loop()`

**for 循环结束时的迭代步进（由 `FOR_END` 和 `CONTINUE` 调用）。**

**逻辑架构：** 根据 `LoopFrame` 中 `values` 的类型判断迭代源类型。

**plain loop 路径（无 values）：**
1. 获取 `lf.start_ip`，直接 `trap.goto(start_ip)` 跳回到循环体开始（实现无限循环）。

**数字迭代路径（source.as_number 成功）：**
1. `values.index += 1.0`。
2. 如果 `index >= end`（达到上限）：调用 `break_for_loop()` 退出。
3. 否则跳回 `start_ip`：
   - 清理（pop）之前创建的子作用域，回收对象内存。
   - 基于父作用域创建新子作用域。
   - 将 `index` 赋值给 `value_id`（当前迭代值）。
4. 返回（循环体即将重新执行）。

**Range 迭代路径（source 是 range 对象）：**
1. 读取 range 的 `end` 和 `step`（step 默认为 1.0）。
2. `values.index += step`。
3. 如果 `index >= end`：`break_for_loop()` 退出。
4. 否则清理旧作用域，创建新作用域：
   - 设置 `value_id` 和 `key_id`（key_id 在 range 中也得到增量值）。
   - `trap.goto(start_ip)` 跳回。

**对象迭代路径（source 是普通对象）：**
1. `values.index += 1.0`。
2. 若 `index >= iter_len`：`break_for_loop()` 退出。
3. 通过 `iter_key_value(obj, index)` 获取当前键值对。
4. 清理旧作用域，创建新作用域：
   - 设置 `value_id`（kv.value）、`index_id`（当前索引）、`key_id`（kv.key）。
   - `trap.goto(start_ip)` 跳回。

**数组迭代路径（source 是数组）：**
1. `values.index += 1.0`。
2. 若 `index >= array_len`：`break_for_loop()` 退出。
3. 通过 `array_index(arr, index)` 获取当前元素。
4. 清理旧作用域，创建新作用域：
   - 设置 `value_id`（元素值）、`index_id`（当前索引）、`key_id`（对于 FOR_2，key_id 得到索引值）。
   - `trap.goto(start_ip)` 跳回。

### `break_for_loop()`

**中断循环（由 `BREAK` 和 `BREAKIFNOT` 调用，以及 `end_for_loop` 在迭代完成时调用）。**

1. 从 `loops` 栈弹出 `LoopFrame`。
2. 调用 `truncate_bases(lp.bases)` 清理作用域和基础帧到循环开始前的状态。
3. 计算跳出位置：`trap.goto(lp.start_ip + lp.jump - 1)`，指向循环结束后的代码。
4. 注意：`lp.jump` 是编译器编码的从循环头到尾的偏移。

### `pop_to_me()`

**将栈顶值弹出并注入当前 `me` 上下文（用于字面量构造过程中的值收集）。**

此方法在 opcodes.rs 的结尾统一处理中也有调用（当 `opargs.is_pop_to_me()`），也在 `handle_pop_to_me()` 中显式调用。

1. 弹出栈顶值。
2. 仅在 `call_has_me()` 时执行以下逻辑：
3. **ID 解析**：如果值是标识符（且未转义），用 `scope_value` 解析为实际变量值。如果值是转义 ID，保持原值（变量引用本身）。
4. 根据 `me` 类型分发：
   - **`ScriptMe::Call { args }`**：调用 `heap.unnamed_fn_arg(args, value)` 作为函数调用的位置参数追加。
   - **`ScriptMe::Object(obj)`**：如果值不是 nil 也不是错误，通过 `heap.vec_push(obj, key, value)` 追加到对象的 vec 中。
   - **`ScriptMe::Pod { pod, offset }`**：调用 `heap.pod_pop_to_me(pod, &mut offset, key, value, ...)` 处理 Pod 字段赋值。更新 `mes` 栈中的 offset 状态。
   - **`ScriptMe::Array(arr)`**：调用 `heap.array_push(arr, value)` 向数组追加元素。
