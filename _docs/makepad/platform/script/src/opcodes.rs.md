# `opcodes.rs` — 操作码主分发函数

## 概述

本文件是 Splash VM 字节码执行器的 **中央分发入口**。它定义了 `ScriptVm::opcode()` 方法，通过 `match` 语句将 `Opcode` 枚举值路由到各个子模块的实现函数。所有的 handler 实现在以下子模块中分别定义：

- `opcodes_ops.rs` — 算术/比较/逻辑操作
- `opcodes_assign.rs` — 赋值操作
- `opcodes_calls.rs` — 函数/方法调用
- `opcodes_control.rs` — 控制流（if/for/return/try/ok）
- `opcodes_vars.rs` — 变量、字段、对象/数组构造
- `opcodes_loops.rs` — 循环迭代辅助函数

---

## 核心方法

### `ScriptVm::opcode(&mut self, opcode: Opcode, opargs: OpcodeArgs)`

这是 VM 执行引擎的主调度方法，接受一个操作码和其参数。每个操作码进入其专用的 handler 方法。

#### 执行流程

1. **无法识别操作码的处理**：如果 `match` 未命中任何操作码（落入通配符 `=>`），执行以下流程：
   - 获取当前指令指针（IP）位置
   - 通过 `self.bx.code.ip_to_loc(ip)` 获取源码位置（文件名、行号等）
   - 打印错误信息：`"UNDEFINED OPCODE {opcode} (raw={n}) at {loc}"`（如有位置信息）或简略版（如无）
   - 调用 `trap.goto_next()` 跳过未定义操作码继续执行

2. **结尾统一处理**：在每个操作码 handler 执行完毕后，如果 `opargs.is_pop_to_me()` 为 true，则调用 `self.pop_to_me()`。`POP_TO_ME_FLAG` 标记由编译器在需要将结果注入当前构造上下文（`me`）时设置。

3. **特殊提前返回的处理**：以下操作码由于改变了控制流或执行上下文，包含 `return` 语句提前退出，跳过结尾的 `pop_to_me` 检查：
   - `CALL_EXEC` / `METHOD_CALL_EXEC`：调用执行后根据 `handle_call_exec` 返回值和 `opargs.is_pop_to_me()` 判断是否调用 `pop_to_me`
   - `RETURN`：函数返回，先从栈弹出返回值（如非 nil），弹出调用帧恢复执行位置
   - `RETURN_IF_ERR`：条件返回，栈顶值是错误则执行返回逻辑
   - `USE`：`use` 语句执行导入操作后直接返回
   - `METHOD_CALL_ARGS`：Pod 类型方法调用参数准备完毕后提前返回

#### 操作码分配明细

**NOP / 算术和位运算（0–12）：**
- `NOP`：空操作，直接 `goto_next()`
- `NOT` / `NEG`：一元运算
- `MUL` / `DIV` / `MOD` / `ADD` / `SUB`：基础算术
- `SHL` / `SHR` / `AND` / `OR` / `XOR`：位运算
- `MOD` 和 `SHL`/`SHR`/`AND`/`OR`/`XOR` 采用泛型 handler 模式

**赋值操作（25–65）：** 所有赋值变体（普通/字段/索引/me/复合）被分发到对应的 `handle_assign_*` 方法。复合赋值如 `ASSIGN_SUB` 被映射到 `handle_f64_scope_assign_op(|a, b| a - b)` 模式，位运算复合赋值映射到 `handle_fu64_*` 变体。

**字符串处理：**
- `CONCAT`：连接两个栈顶值为字符串

**比较与逻辑（14–24）：**
- `EQ` / `NEQ`：调用 `heap.deep_eq` 做深度相等性比较
- `LT` / `GT` / `LEQ` / `GEQ`：采用 `handle_f64_cmp_op` 泛型 handler
- `LOGIC_AND_TEST` / `LOGIC_OR_TEST` / `NIL_OR_TEST`：短路求值测试
- `SHALLOW_EQ` / `SHALLOW_NEQ`：基于 `ScriptValue` 的 `==` 运算符做浅比较

**对象/数组字面量（66–72、118–125）：**
- `BEGIN_PROTO` / `END_PROTO`：原型对象构建
- `PROTO_INHERIT_READ` / `PROTO_INHERIT_WRITE`：继承读取/写入
- `SCOPE_INHERIT_READ` / `SCOPE_INHERIT_WRITE`：作用域继承
- `FIELD_INHERIT_READ` / `FIELD_INHERIT_WRITE`：字段继承
- `INDEX_INHERIT_READ` / `INDEX_INHERIT_WRITE`：索引继承
- `BEGIN_BARE` / `END_BARE`：裸对象构建
- `BEGIN_ARRAY` / `END_ARRAY`：数组构建

**函数定义与调用（73–83）：**
- `CALL_ARGS`：准备函数调用参数对象
- `CALL_EXEC` / `METHOD_CALL_EXEC`：调用执行（合并处理）
- `METHOD_CALL_ARGS`：方法调用参数与动态分发
- `FN_ARGS` / `FN_LET_ARGS`：函数参数定义
- `FN_ARG_DYN` / `FN_ARG_TYPED`：单个参数（动态/类型化）
- `FN_BODY_DYN` / `FN_BODY_TYPED`：函数体定义
- `RETURN`：函数返回

**变量/字段（86–91、115–117）：**
- `FIELD` / `FIELD_NIL` / `ME_FIELD` / `PROTO_FIELD`：字段读取的各种变体
- `POP_TO_ME` / `ME_SPLAT`：上下文操作
- `ARRAY_INDEX`：数组/对象索引读取
- `LET_DYN` / `LET_TYPED` / `VAR_DYN` / `VAR_TYPED`：变量声明
- `USE`：模块/对象导入

**上下文（94–98）：**
- `SEARCH_TREE`：属性树搜索
- `LOG`：日志输出
- `ME` / `SCOPE`：获取当前上下文/作用域

**循环（99–107）：**
- `FOR_1` / `FOR_2` / `FOR_3`：三种 for 循环变体
- `LOOP`：无限循环
- `FOR_END` / `BREAK` / `BREAKIFNOT` / `CONTINUE`：循环控制
- `RANGE`：range 对象创建

**类型检查与错误处理（108–114）：**
- `IS`：`is` 类型检查关键字
- `RETURN_IF_ERR`：错误传播 `?`
- `TRY_TEST` / `TRY_ERR` / `TRY_OK`：try 块
- `OK_TEST` / `OK_END`：ok 块

**解构（126–129）：**
- `DUP` / `DROP`：栈操作
- `ARRAY_INDEX_NIL`：安全的数组索引
- `LET_DESTRUCT_ARRAY_EL` / `LET_DESTRUCT_OBJECT_EL`：解构赋值
