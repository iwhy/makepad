# shader_control.rs — 控制流编译

**源码路径:** `platform/script/src/shader_control.rs`  
**总行数:** 655 行  
**核心职责:** 将 Splash 字节码中的控制流指令（if/else、for 循环、while 循环、return）编译为目标后端着色器代码。

---

## 核心架构

控制流编译的核心在于 **`self.mes`（Meta Expression 栈）** 上保存的控制流状态标记。每个 `if` 块被表示为 `ShaderMe::IfBody` 标记，其中记录了：
- `target_ip`: 该 if 块结束处的字节码 IP（用于判断何时关闭该块）
- `start_pos`: if 开始处 `self.out` 字符串的偏移量（用于在 if 块之前插入 phi 变量声明）
- `phi`: if-else 表达式的 phi 变量名（如 `_phi_123`），用于保存 if/else 的最终值
- `has_return`: 当前分支是否已 return
- `if_branch_returned`: if 分支是否已 return（对 else 分支而言）
- `phi_assigned_by_inner`: 是否由内部嵌套 if 块已赋值给此 phi

---

## 方法详解

### `is_unreachable` — 检查是否处于不可达代码中（第17-36行）

从 `self.mes` 栈顶向下遍历，检测是否进入了已 return 的作用域。如果遇到 `IfBody { has_return: true }` 或 `FnBody { escaped: true }`，返回 `true`。此方法用于在不可达代码中生成跳转指令跳过不必要的分支。

### `is_parent_scope_unreachable` — 检查父作用域是否不可达（第40-61行）

与 `is_unreachable` 类似，但跳过最内层的 `IfBody`。用于 IF_ELSE 指令判断是否需要生成 else 分支代码——如果最内层 if 之前就已经不可达，则分支代码已无用。

### `find_and_mark_outer_phi` — 查找并标记外部 phi 变量（第67-89行）

跳过最内层 IfBody，查找外层 IfBody 的 phi 变量。如果找到，标记该外层 phi 为 `phi_assigned_by_inner = true`，并返回 phi 变量名。用于 match/else-if 链中内层 if 块没有自己的 phi 但需要将值赋给外层 phi 的情况。

### `handle_if_else_phi` — if/else phi 归并处理（第91-253行）

这是一个循环方法（loop），处理所有 `target_ip` 已到达的 IfBody。这是控制流编译中最复杂的部分，处理 if-else 表达式的结果合并：

1. **检查是否需要处理**（第96-103行）：检查 ME 栈顶是否为 IfBody 且其 `target_ip <= self.trap.ip.index`，如果是则继续，否则 break。

2. **两分支均 return 检测**（第120行）：如果 `if_branch_returned && has_return` 都为 true，说明两个分支都已 return，后续代码成为不可达代码。

3. **从 else 分支弹出值**（第130-189行）：如果栈上还有值说明 else 分支产生了一个值：
   - 如果值是 void：作为语句输出（追加 `;\n`）
   - 如果值非 void 且有 phi 变量：写入 `phi = value;\n`，并在 `start_pos` 处插入 phi 变量的声明（带零初始化）
   - 如果值非 void 但无 phi（match/else-if 链场景）：通过 `find_and_mark_outer_phi` 查找外层 phi 并赋值

4. **else 分支无值但 if 分支有 phi**（第191-223行）：if 分支产生了 phi 但 else 分支没有。此时在 `start_pos` 处插入 phi 声明（零初始化）。如果内层代码已赋值给此 phi（`phi_assigned_by_inner`），则将 phi 值推到栈上。

5. **只有 return 无值**（第224-228行）：if 分支有 return 但 else 分支既无值也无 phi。设置 `skip_next_pop_to_me = true` 以防止后续 POP_TO_ME 导致栈下溢。

6. **关闭作用域**（第230-232行）：输出 `}\n`、调用 `shader_scope.exit_scope()`、弹出 ME。

7. **逃逸传播**（第234-248行）：如果两个分支都 return 了，将逃逸状态传播到父作用域（父 IfBody 或 FnBody）。

### `handle_if_test` — if 测试代码生成（第256-276行）

弹出条件值，生成 `if(condition){\n` 代码。记录当前 `self.out` 在 `start_pos` 处，压入 `ShaderMe::IfBody` 标记并进入新的变量作用域。

### `handle_if_test_unreachable` — 不可达代码中的 if 测试（第280-295行）

当已经处于不可达代码中时，不生成任何代码也不弹出栈，仅压入一个 `IfBody` 标记（`has_return: true`, `created_unreachable: true`）用于跟踪控制流结构，使得后续的 IF_ELSE 和 IF_ELSE_PHI 能正确关闭。

### `handle_if_else` — else 分支代码生成（第297-361行）

1. 检查 if 分支是否在栈上产生了值（`self.stack.types.len() > stack_depth`），如果是则弹出该值
2. 判断值是否为 void，如果不是则创建 phi 变量（格式 `_phi_{start_pos}`），写入 `phi = value;\n`
3. 输出 `}\nelse{\n`，退出 if 作用域并进入 else 作用域
4. 更新 `target_ip` 为新的结束 IP
5. 将 `has_return` 保存到 `if_branch_returned` 后重置为 false

### `handle_if_else_unreachable` — 不可达代码中的 else 处理（第364-378行）

仅更新 `target_ip` 和 `if_branch_returned`，不生成任何代码。

### `handle_if_else_phi_unreachable` — 不可达代码中的 phi 归并（第381-418行）

仅当 `created_unreachable` 为 false 时才输出 `}\n`（因为如果从未生成过 `if(...){`，就不需要对应的 `}`）。处理两分支都 return 时的逃逸传播。

### `handle_return` — return 语句代码生成（第420-515行）

1. **已逃逸检测**（第427-445行）：如果函数体已全部 return 返回，仅消费栈上返回值（如有）后直接返回。

2. **返回值解析**（第452-462行）：调用 `pop_resolved` 弹出并具体化返回值类型。void 返回创建一个空字符串。

3. **记录返回类型**（第465-493行）：在 ME 栈中找到 `FnBody` 标记，设置其 `ret` 为返回类型。生成 `return value;\n` 或 `value;\nreturn;\n`（void 类型）。

4. **IfBody 标记**（第498-507行）：在 ME 栈中找到最内层 IfBody，设置 `has_return = true`，使编辑器知道此分支已 return。

5. **不设置 Trap**（第509-514行）：与解释器不同，编译器不需要设置 `ScriptTrapOn::Return`。编译器必须继续处理后续所有字节码以正确关闭 if/else 块等控制结构，`compile_fn` 依靠 `fn_end_index` 而不是 return trap 来知道何时停止。

### `handle_for_1` — for 循环初始化（第517-585行）

弹出范围和循环变量信息：
1. 范围类型必须是 `ShaderType::Range { start, end, ty }`
2. 循环变量必须是 `ShaderType::Id`
3. **类型校验**：Shder 的 for 循环只支持 `u32` 类型的范围。如果范围类型是 `i32` 则自动转为 `u32`
4. 生成循环初始化和头部代码：
   - WGSL: `for(var var: u32 = start; var < end; var++){`
   - Rust: `for var in start..end {`
   - GLSL: `for(u32 var = u32(start); var < u32(end); var++){`（GLSL ES 3.0 不允许隐式 int→uint 转换）
   - Metal/HLSL: `for(u32 var = start; var < end; var++){`
5. 进入新的 scope 并压入 `ShaderMe::ForLoop` 标记

### `handle_for_end` — for 循环结束（第587-601行）

弹出 `ShaderMe::ForLoop` 或 `ShaderMe::LoopBody` 标记，输出 `}\n`，退出 scope。

### `handle_loop` — while(true) 循环（第603-608行）

输出 `while(true){\n`，进入新的 scope，压入 `ShaderMe::LoopBody` 标记。

### `handle_break` — break 语句（第611-613行）

输出 `break;\n`。

### `handle_breakifnot` — 条件 break（第615-618行）

弹出条件值，输出 `if(!(condition)){break;}\n`。

### `handle_continue` — continue 语句（第620-622行）

输出 `continue;\n`。

### `handle_range` — 范围表达式构造（第624-654行）

弹出起始值和结束值，对两者调用 `make_concrete` 确保是数值类型。校验两者都是数值类型后，构造 `ShaderType::Range { start, end, ty }` 推回栈上。如果不是数值类型则报错。
