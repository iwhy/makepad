# `opcodes_calls.rs` — 函数/方法调用与函数定义操作码执行（441 行）

## 概述

本文件实现了 Splash VM 中**函数调用、方法调用、函数定义**相关操作码的执行逻辑。包括调用参数准备、调用执行、动态方法分发、函数参数处理、函数体定义等。

---

## 方法清单与实现逻辑

### 调用处理

#### `handle_call_args()`

**函数调用参数准备阶段。**

1. 从栈弹出函数对象并解析。
2. **Pod 类型路径**：如果函数是 Pod 类型（`heap.pod_type(fnobj)` 返回 Some），则通过 `heap.new_pod(ty)` 创建 Pod 实例：
   - 将 `ScriptMe::Pod { pod, offset }` 压入 `mes` 栈，用于后续参数收集。
3. **对象/函数路径**：否则通过 `heap.new_with_proto(fnobj)` 基于函数原型创建参数对象：
   - 调用 `heap.clear_object_deep(args)` 清空深层属性。
   - 将 `ScriptMe::Call { args, sself: None, method: None }` 压入 `mes` 栈，等待后续参数通过 `pop_to_me` 机制填充。
4. `goto_next()`。

#### `handle_call_exec(opargs) -> bool`

**函数调用执行阶段。**

1. 从 `mes` 栈弹出 `me` 上下文：
   - **Pod 路径**：检查参数总数（`pod_check_arg_total`），将 Pod 实例压栈作为结果。返回 `true`（需要 caller 处理 `pop_to_me`）。
   - **Call 路径**：提取 `args`、`sself`（self 参数）和 `method`（方法名）。
2. 如果存在 `sself`：通过 `force_value_in_map(args, id!(self).into(), sself)` 将 `self` 参数强行写入参数对象。
3. 分别调用 `set_object_deep(args)` 和 `set_object_storage_auto(args)` 最终确定参数对象。
4. **动态分发路径**（有 `method` 但无函数原型）：
   - 通过 `sself.value_type()` 获取接收者类型索引。
   - 查询 `native.calls` 表中的 `call` 回调指针。
   - 如果有回调：暂停当前线程（`is_paused = true`），通过 unsafe 指针调用回调函数。
     - **暂停检测**：回调后如果 `trap.on` 被设为 `Pause`，重新压入 `ScriptMe::Call` 到 `mes` 并返回 `false`，后续可恢复执行。
     - **正常完成**：恢复线程（`is_paused = false`），将返回值压栈，释放 args 对象，返回 `true`。
   - 如果没有 call handler：生成 `script_err_not_found!` 错误。
5. **标准函数路径**（有函数原型 `parent_as_fn`）：
   - **Native 函数**：从 `native.functions` 获取函数指针，暂停线程后通过 unsafe 指针执行。类似地支持暂停/恢复机制。完成后释放 args 对象，压栈返回值。
   - **Script 函数**：构造 `CallFrame`（包含 `bases`、`args`、`return_ip`）。将 args 压入 `scopes` 栈作为新作用域，将 call 压入 `calls` 栈，设置 `trap.ip` 跳转到函数入口。返回 `false`（由后续 `RETURN` 处理 `pop_to_me`）。
6. 如果目标不是函数，生成错误值。

**返回值含义**：`true` → caller 需要处理 `pop_to_me`；`false` → 跳过。

#### `handle_method_call_args() -> bool`

**方法调用参数准备阶段（支持动态分发）。**

1. 从栈弹出方法名和方法接收者 `sself`。
2. 在接收者上查找方法函数：
   - **对象方法**：`heap.object_method(sself.as_object(), method)`
   - **Pod 方法**：`heap.pod_method(sself.as_pod(), method)`
3. 如果方法找到且不是 nil/err：
   - 如果是 Pod 类型方法：创建 Pod 实例，压入 `ScriptMe::Pod` 到 `mes`，返回 `true`（跳过 `pop_to_me`）。
   - 否则：`heap.new_with_proto(fnobj)` 创建参数对象。
4. 如果方法未找到：
   - 查询类型系统的 `type_table` 和方法表 `calls`。
   - **常规方法路径**：如果方法注册在 type_table 中，创建参数对象。
   - **动态分发路径**：如果类型有 `calls` 回调但没找到具体方法：
     - 创建空的 `vec2` 类型参数对象。
     - 压入 `ScriptMe::Call { args, sself: Some(sself), method: Some(method) }`（携带 method 名）。
     - 返回 `false`，后续 `handle_call_exec` 将进行动态分发。
   - **方法不存在路径**：生成 `script_err_not_found!` 错误，创建 `undefined_function` 对象。
5. 对参数对象调用 `clear_object_deep`。
6. 压入 `ScriptMe::Call { args, sself: Some(sself), method: None }` 到 `mes`。
7. 返回 `false`。

**返回值含义**：`true` → Pod 路径，caller 应提前返回；`false` → 正常路径。

---

### 函数定义处理

#### `handle_fn_args()`

**动态函数参数列表开始。**

1. 获取当前作用域链的顶层 `scope`。
2. 基于该 scope 创建新的参数对象：`heap.new_with_proto(scope.into())`。
3. 设置存储类型为 `vec2`（有序命名参数）：`set_object_storage_vec2`。
4. 清空深层属性：`clear_object_deep`。
5. 将 `ScriptMe::Object(me)` 压入 `mes` 栈，后续参数通过 `pop_to_me` 填充。
6. `goto_next()`。

#### `handle_fn_let_args()`

**`let` 函数参数列表开始（会绑定到作用域）。**

1. 从栈弹出函数名标识符。
2. 获取当前顶层 scope，基于 scope 创建参数对象（与 `handle_fn_args` 相同）。
3. 将 `ScriptMe::Object(me)` 压入 `mes`。
4. 同时通过 `def_scope_value` 将参数对象绑定到指定标识符的作用域。
5. `goto_next()`。

#### `handle_fn_arg_dyn(opargs)`

**动态类型单个函数参数处理。**

1. 从 `opargs` 判断是否有初始值：若 `is_nil()` 则为 `NIL`，否则从栈弹出。
2. 从栈弹出参数名标识符。
3. 获取当前 `mes` 上下文的最后一个（应为 `ScriptMe::Object`）：
   - 如果是 `Object(obj)`：若参数名为 `self` 且参数对象尚未有元素，则忽略（跳过 `self` 保留位置）；否则通过 `heap.set_value(*obj, id, value)` 将参数设置到对象中。
   - 如果是 `Call` / `Array` / `Pod`：生成错误（非法上下文）。
4. `goto_next()`。

#### `handle_fn_arg_typed(opargs)`

**类型标注的单个函数参数处理。**

与 `handle_fn_arg_dyn` 类似，但多了一个类型参数：
1. 从栈弹出值（或 nil）、类型值、参数名。
2. 对 `Object(obj)` 上下文：调用 `heap.set_value(*obj, id.into(), ty)` 将**类型**（而非值）设置到参数对象中。实际值在后续通过其他方式传递。
3. `goto_next()`。

#### `handle_fn_body_dyn(opargs)`

**动态函数体定义。**

1. `opargs.to_u32()` 获取跳过整个函数体的跳转偏移。
2. 从 `mes` 栈弹出上一个上下文（应为参数收集阶段的 `Object(me)`）：
   - 如果是 `Object(obj)`：
     - 使用 `heap.set_fn(obj, ScriptFnPtr::Script(ScriptIp{...}))` 将当前指令位置作为函数入口记录到对象中。
     - 将对象的引用压栈（作为函数值）。
   - 如果是其他类型：生成错误，压入 `NIL`。
3. 调用 `trap.goto_rel(jump_over_fn)` 跳过函数体，指向后续代码。
4. 如果 `mes` 为空，生成错误。

#### `handle_fn_body_typed(opargs)`

**类型标注函数体定义。**

与 `handle_fn_body_dyn` 逻辑相同，但额外从栈弹出返回类型值（`_return_type`）：
1. 弹出返回类型（当前忽略，仅用于类型信息传递）。
2. 弹出 `me`，如果是 `Object(obj)`：设置函数指针并压栈。
3. 跳转到函数体之后。
4. 如果 `mes` 为空，生成错误。
