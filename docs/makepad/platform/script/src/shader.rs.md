# shader.rs — 着色器 IR（中间表示）与编译器核心

**文件路径**: `platform/script/src/shader.rs`  
**行数**: 1693 行  
**作用**: Makepad 脚本虚拟机的着色器编译器前端。将 VM 字节码（IR 指令流）翻译为着色器源代码字符串。定义类型系统、作用域管理、控制流栈、以及所有字节码指令的着色器代码生成处理函数。

---

## 主要类型定义

### `ShaderPodArg`（第 40–44 行）
表示着色器 Pod 构造函数的单个参数，包含可选的参数名（用于命名参数）、ShaderType 类型信息和字符串表达式。

### `ShaderMe`（第 47–111 行）
着色器编译器维护的"元执行栈"（Meta Execution Stack），用于跟踪控制流嵌套结构。每个变体对应一种代码生成上下文：

- **`FnBody`**: 函数体上下文。记录返回值类型 `ret`、是否所有代码路径都已返回 `escaped`、以及栈深度 `stack_depth`，用于编译完成后恢复栈。
- **`LoopBody`**: 循环体上下文（`while true`），只记录栈深度。
- **`ForLoop`**: for 循环上下文，记录循环变量的 LiveId 和栈深度。
- **`IfBody`**: if/else 分支上下文。包含跳转目标 IP、输出字符串位置、phi 节点变量（用于合并 if-else 分支的值）、phi 类型、以及多个布尔标记（`has_return` 当前分支是否有 return、`if_branch_returned` if 分支是否返回、`phi_assigned_by_inner` 是否被内部 if 赋值、`created_unreachable` 创建时是否已不可达）。
- **`LogicOp`**: 逻辑操作（`&&`、`||`）上下文。包含跳转目标、操作符字符串、第一操作数表达式及其类型。着色器需要同时求值两个操作数后合并。
- **`BuiltinCall`**: 内置函数调用上下文。记录函数名 LiveId、NativeId 函数指针、以及参数列表（类型+表达式）。
- **`PodBuiltinMethod`**: Pod 类型内置方法调用上下文（如 `x.mix(y, a)`），记录方法名、self 的 Pod 类型和参数。
- **`ScriptCall`**: 脚本函数调用上下文。包含输出字符串累加器、函数名、函数对象、self 类型、参数类型列表。
- **`Pod`**: Pod 结构体构造上下文。记录 Pod 类型和参数列表。
- **`ArrayConstruct`**: 固定长度数组构造上下文。记录元素表达式和元素类型。
- **`TextureBuiltin`**: 纹理内置方法调用上下文。记录方法 ID、纹理类型、纹理表达式和参数。

### `ShaderType`（第 113–146 行）
着色器类型系统，区分编译期抽象类型和运行时具体类型：

- **`None`**: 无类型
- **`IoSelf(ScriptObject)`**: `self` 引用为 IO 对象
- **`PodType(ScriptPodType)`**: 仅类型信息（用于类型注解）
- **`Pod(ScriptPodType)`**: 具体 Pod 值
- **`PodPtr(ScriptPodType)`**: Pod 指针（Metal 统一缓冲区）
- **`Texture(TextureType)`**: 纹理类型
- **`Id(LiveId)`**: 未解析的标识符，需要查作用域
- **`AbstractInt`/`AbstractFloat`**: 字面量抽象类型（未确定具体是 f32 还是 f16）
- **`Range { start, end, ty }`**: 范围类型，用于 for 循环
- **`Error(ScriptValue)`**: 错误类型
- **`ScopeObject(ScriptObject)`**: 脚本作用域对象
- **`ScopeUniformBuffer { obj, pod_ty }`**: 统一缓冲区
- **`ScopeTexture { obj, tex_type, shader_name }`**: 作用域纹理

#### `ShaderType::make_concrete`（第 148–167 行）
将抽象类型转换为具体 PodType。`AbstractInt` → `pod_i32`，`AbstractFloat` → `pod_f32`，`Range` 返回其内部元素类型。纹理和错误返回 None。

### `ShaderScopeItem`（第 170–177 行）
作用域中的单一条目，可以是：
- `IoSelf(ScriptObject)`: `self` 引用为 IO
- `ScopeObject(ScriptObject)`: `self` 为作用域对象
- `Param { ty, shadow }`: 函数参数
- `Let { ty, shadow }`: `let` 绑定
- `Var { ty, shadow }`: `var` 可变绑定
- `PodType { ty, shadow }`: 类型别名

每个变体都提供 `ty()` 方法返回 PodType，`shadow()` 方法返回影子计数。

### `ShaderScope`（第 204–206 行）
嵌套作用域栈，使用 `Vec<LiveIdMap<LiveId, ShaderScopeItem>>`。每层是一个 LiveId 到 ShaderScopeItem 的映射。

### `ShaderFnCompiler`（第 209–219 行）
着色器函数编译的核心结构体：
- `out`: 输出的着色器源代码字符串
- `stack`: 表达式计算栈（ShaderStack）
- `script_scope`: 脚本作用域对象
- `shader_scope`: 着色器作用域
- `mes`: 元执行栈（ShaderMe）
- `trap`: 错误处理
- `debug`: 调试标志
- `skip_next_pop_to_me`: 用于跳过已 return 的 if 后的 pop

### `ShaderStack`（第 222–227 行）
表达式栈，包含类型向量 `types` 和字符串表达式向量 `strings`，以及回收池 `free`。

---

## 工具宏

### `push_fmt!`（第 229–234 行）
格式化并推入栈顶。从回收池取字符串，write! 格式化后 push。

### `free_fmt!`（第 236–242 行）
仅格式化返回字符串，不推栈。也使用回收池。

---

## `write_shader_float`（第 23–37 行）
将 f64 输出为着色器可读的浮点文本。对极大（≥1e15）或极小（<1e-6）的数使用科学计数法，避免 Metal 解析器截断。确保输出总是包含小数点以便后续追加 `f` 后缀。

---

## `ShaderScope` 方法实现

### `enter_scope` / `exit_scope`（第 245–251 行）
进入/退出嵌套作用域：push/pop `shader_scope` 向量。

### `find_var`（第 253–260 行）
从内向外遍历作用域栈查找标识符，返回 `(ShaderScopeItem, shadow)` 元组。反向迭代保证内层作用域优先。

### `all_var_names`（第 263–273 行）
收集所有作用域层中的变量名，用于自动补全建议。去重后返回。

### `define_io_self` / `define_scope_object`（第 275–283 行）
在当前作用域注册 `self` 的 IO 或 ScopeObject 类型。

### `define_var` / `define_let` / `define_param` / `define_pod_type`（第 285–329 行）
在当前作用域定义变量。如果已存在同名变量（影子作用域），递增 shadow 计数器。每种定义方法对应不同的 ShaderScopeItem 变体。

---

## `ShaderStack` 方法实现

### `pop`（第 333–340 行）
弹出栈顶的 `(ShaderType, String)`。栈下溢时报告错误。

### `peek`（第 342–350 行）
查看栈顶但不弹出。下溢时返回静态空值并报错。

### `push`（第 352–359 行）
推入 `(ShaderType, String)`。超过 `stack_limit` 时报栈溢出错误。

### `new_string` / `free_string`（第 361–374 行）
字符串回收池。`new_string` 从回收池取空字符串（减少分配），`free_string` 清空后归还。

---

## `ShaderFnCompiler` 方法实现

### `shader_math_const_value`（第 377–390 行）
将数学常量 LiveId 映射为 f64 值：PI、E、LN2、LN10、LOG2E、LOG10E、SQRT1_2、TORAD、GOLDEN。这些常量在着色器编译期被内联为字面量，不走统一缓冲区。

### `new`（第 392–405 行）
构造 ShaderFnCompiler。初始化栈限制为 1000000，创建第一层空作用域，其余字段使用 Default。

### `compile_fn`（第 407–519 行）
着色器函数编译入口。核心过程：
1. 调用 `backend.register_ids()` 注册后端特有类型名。
2. 创建 FnBody 元栈帧，记录空返回值。
3. 从 `FN_BODY_DYN` 操作码中计算函数结束位置 `fn_end_index`。
4. 主循环：从 `trap.ip` 位置开始逐指令向前遍历，直到 `fn_end_index`。
5. 每次迭代重新借用字节码（允许方法调用中可变借用 vm）。
6. 如果设置了 `skip_next_pop_to_me` 且下一条是 POP_TO_ME，则跳过处理不可达代码。
7. 不可达代码时仅处理控制流操作码（IF_TEST、IF_ELSE）以维护结构完整性；非不可达时执行完整编译。
8. 在每条指令前后调用 `handle_logic_phi` 和 `handle_if_else_phi` 处理短路逻辑和 if-else 合并。
9. 处理编译错误日志输出。
10. 函数结束时弹出 FnBody 帧，返回返回值类型（默认为 void）。

### `pop_resolved`（第 521–752 行）
从栈弹出值并解析标识符。关键路径：
1. 弹出 `(ShaderType, String)`。
2. 如果是 `ShaderType::Id(id)`，先在着色器作用域查找。
3. 找到则返回 `ShaderType::Pod(ty)`，变量名由 `backend.map_param_name`/`map_local_name` 映射。
4. 未找到则在脚本作用域（`script_scope`）中查找。
5. 如果是 `shader_io` 对象，根据 IO 类型分类处理：
   - `SHADER_IO_UNIFORM_BUFFER` → 返回 `ShaderType::ScopeUniformBuffer`
   - 各种纹理类型 → 返回 `ShaderType::ScopeTexture`，并注册到 `output.scope_textures` 和 `output.io`
   - 其他 `shader_io` 类型 → 报错
6. 如果是普通对象 → 返回 `ShaderType::ScopeObject`
7. 如果是数学常量（PI 等）→ 内联为字面量
8. 如果是普通值 → 添加为 ScopeUniform，注册到 `output.scope_uniforms` 和 `output.io`
9. 所有情况使用 `ShaderIoPrefix` 生成正确的引用前缀（与后端相关）

### `get_scope_value_pod_type`（第 755–774 行）
从脚本值获取对应的 Pod 类型。支持：基本类型（f32/f64/bool 等）、颜色（→ vec4f）、Pod 实例。

### `generate_scope_uniform_name`（第 778–838 行）
为作用域统一变量生成唯一 LiveId。处理名称冲突的三种策略：
1. 直接使用基本名称（未冲突时）
2. 追加 `_objN`（N 为对象索引）
3. 追加 `_objN_M` 计数器后缀

### `generate_scope_uniform_buffer_names`（第 845–892 行）
为作用域统一缓冲区生成 `(shader_name, struct_type_name)` 对。格式：`scopebuf_{index}` 和 `IoScopeUniformBuf{index}`。冲突时追加计数器。

### `generate_scope_texture_name`（第 896–963 行）
为作用域纹理生成唯一名称。策略同 `generate_scope_uniform_name`，冲突时使用 `_objN` 后缀。

### `push_immediate`（第 965–1059 行）
将立即数（字面量）值推入表达式栈。处理多种脚本值类型：
- **f64**: 抽象浮点（未定具体类型），使用 `write_shader_float` 格式化
- **u40**: 抽象整数
- **LiveId**: 标识符引用，若在作用域中找到则映射名称，否则直接格式化
- **f32**: 具体浮点，追加 `f`（Metal/GLSL）或 `f32`（Rust）
- **f16**: half 类型，追加 `h`（Metal）或 `f32`（Rust）
- **u32**: 无符号整数，追加 `u`（Metal/GLSL）或 `u32`（Rust）
- **i32**: 有符号整数，追加 `i`（Metal/GLSL）或 `i32`（Rust）
- **bool**: 直接输出 `true`/`false`
- **color**: 展开为 `vec4f(r, g, b, a)` 构造函数调用

### `ensure_struct_name`（第 1061–1090 行）
确保 Pod 结构体有名称记录。检查名称一致性（允许 `f32`↔`float`、`u32`↔`uint`、`i32`↔`int` 别名），将结构体类型插入 `output.structs`。

### `opcode`（第 1092–1545 行）
着色器编译器的主要指令分发器。将每个字节码操作码映射到对应的 `handle_*` 方法。分类处理：

**算术指令**：NOT、NEG、MUL、DIV、MOD、ADD、SUB、SHL、SHR、AND、OR、XOR → 调用 `handle_arithmetic`/`handle_not`/`handle_neg`

**赋值指令**：
- `ASSIGN` → `handle_assign`（直接赋值）
- `ASSIGN_ADD/SUB/MUL/DIV/MOD` → `handle_arithmetic_assign`（复合赋值）
- `ASSIGN_AND/OR/XOR/SHL/SHR` → 同上的位运算版本
- `ASSIGN_FIELD*` → `handle_assign_field`/`handle_arithmetic_field_assign`（字段赋值）
- `ASSIGN_INDEX*` → `handle_assign_index`/`handle_arithmetic_index_assign`（索引赋值）
- `ASSIGN_ME` → `handle_assign_me`（声明性赋值 `:=`）
- `ASSIGN_ME_VEC/BEFORE/AFTER/BEGIN` → 不支持的着色器特性报错

**比较指令**：
- `EQ、NEQ、LT、GT、LEQ、GEQ` → `handle_eq`（比较运算）
- `SHALLOW_EQ/SHALLOW_NEQ` → 不支持

**逻辑指令**：
- `LOGIC_AND_TEST、LOGIC_OR_TEST` → `handle_logic_test`（短路逻辑）
- `NIL_OR_TEST` → 不支持

**对象/数组**：
- `BEGIN_PROTO/END_PROTO、BEGIN_BARE/END_BARE、BEGIN_ARRAY/END_ARRAY` → 不支持。数组建议使用固定大小数组

**调用指令**：
- `CALL_ARGS` → `handle_call_args`
- `CALL_EXEC/METHOD_CALL_EXEC` → `handle_call_exec`
- `METHOD_CALL_ARGS` → `handle_method_call_args`
- `FN_ARGS/FN_LET_ARGS/FN_ARG_DYN/FN_ARG_TYPED/FN_BODY_DYN/FN_BODY_TYPED` → 不支持嵌套函数

**控制流**：
- `IF_TEST → handle_if_test`、`IF_ELSE → handle_if_else`
- `FOR_1 → handle_for_1`、`FOR_2/FOR_3` → 不支持
- `LOOP → handle_loop`、`FOR_END → handle_for_end`
- `BREAK → handle_break`、`BREAKIFNOT → handle_breakifnot`
- `CONTINUE → handle_continue`
- `RETURN → handle_return`

**其他**：
- `FIELD、PROTO_FIELD → handle_field`（字段访问）
- `ARRAY_INDEX → handle_array_index`
- `LET_DYN → handle_let_dyn`、`VAR_DYN → handle_var_dyn`
- `RANGE → handle_range`（范围字面量 `a..b`）
- `LOG → handle_log`（调试日志）
- `POP_TO_ME → pop_to_me`（弹出计算值给当前 ME）
- 每个指令后调用 `maybe_pop_to_me` 处理可选弹出

不支持的指令包括：字符串连接、继承操作、空值合并、类型检查、错误传播、树搜索、use 导入等。

### `pop_to_me`（第 1547–1681 行）
将栈顶值弹出并传递当前 ME（元执行栈帧）。按 ME 类型不同处理：
- **FnBody/ForLoop/LoopBody/IfBody**: 弹出值作为表达式语句追加到 `self.out`（加 `;` 和换行）
- **Pod**: 作为构造参数收集，检测命名参数和位置参数的混用错误
- **ArrayConstruct**: 检查数组元素类型一致性，首元素确定类型
- **TextureBuiltin**: 收集参数表达式
- **ScriptCall**: 收集参数类型和表达式。对 Rust 后端特殊处理：如果嵌套函数调用同时借用 `rcx`，将其提升为 let 绑定以避免双重可变借用
- **BuiltinCall/PodBuiltinMethod**: 收集参数类型，解析 Id 为具体 PodType

### `maybe_pop_to_me`（第 1683–1692 行）
如果操作码的 opargs 标记了 `is_pop_to_me()`（通过参数索引指示），调用 `pop_to_me`。这是操作码编码中常见的后缀弹出模式。

---

## 文件职责总结

`shader.rs` 是 Makepad 着色器编译器的 IR 核心，职责包括：

1. **类型系统**: 定义着色器中的类型表示（抽象类型、具体 Pod 类型、纹理类型、作用域引用等）
2. **作用域管理**: 嵌套作用域的进入/退出、变量定义和查找
3. **表达式栈**: 管理编译过程中的表达式计算栈，带回收池优化
4. **元执行栈**: 控制流结构（函数体、循环、if-else、逻辑短路、函数调用、构造等）的上下文管理
5. **指令编译**: 将 VM 字节码的每条指令翻译为对应后端着色器语言的源代码字符串
6. **作用域引用解析**: 脚本作用域中的值和纹理/缓冲区的自动导入和命名管理
