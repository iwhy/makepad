# `parser.rs` 源码解读

**路径:** `platform/script/src/parser.rs`
**行数:** 4265
**核心职责:** Splash 脚本语言的递归下降解析器，以**状态机驱动**的方式将 Token 流解析为 Opcode 字节码指令序列。不生成 AST，直接边解析边发射 Opcode。

---

## 设计哲学

Splash 解析器采用**基于状态栈的操作器优先分析法（Operator-Precedence Parser）**，而非经典的递归下降函数调用模式。整解析器仅有一个核心入口函数 `parse_step()`，通过一个 `Vec<State>` 状态栈驱动所有解析动作。

**工作流程:**
1. `parse()` 主循环不断从 tokenizer 读取 Token
2. 每读一个 Token，从状态栈 `pop` 栈顶状态，与当前 Token 的类别/操作符/标识符进行匹配
3. 根据匹配结果：发射 Opcode 到 `opcodes` 向量，或向状态栈 `push` 新的状态（相当于递归调用）
4. 返回 `step` 值（0 或 1），指示是否消耗当前 Token
5. 当状态栈为空时解析结束

**操作符优先级:**
```
1  Identifier path (最高优先级)
2  Method calls (.)
3  Field expression (.)
4  Function call, array index
5  ? (return if err)
6  unary - ! * borrow
7  as
8  * / %
9  + -
10 << >>
11 &
12 ^
13 |
14 == != < > <= >=   (以及 ++, -:, is)
15 &&                (以及 ===, !==)
16 ||
17 .. (range)
18 = := <: >: ^: +: += -= *= /= %=
19 &= |= ^= <<= >>= ?= (最低优先级)
```

---

## 类型定义

### `State` 枚举 (行 14–452)

解析器的**核心状态机枚举**，定义了所有可能的解析上下文，每个变体携带该状态所需的元数据。

| 状态变体 | 含义 |
|---------|------|
| `BeginStmt` | 期望一个语句的开始（表达式/声明/控制流） |
| `BeginExpr` | 期望一个表达式，携带 `required` 标记是否必填 |
| `EndExpr` | 表达式结束，检查后续操作符或括号 |
| `EndStmt` | 语句结束，处理 POP_TO_ME 或跳转 |
| `EscapedId` | `@` 转义标识符后等待标识符 |
| `ForIdent`/`ForBody`/`ForExpr`/`ForBlock` | `for` 循环各阶段（标识符收集、范围表达式、主体） |
| `Loop`/`While`/`WhileTest` | `loop`/`while` 循环 |
| `IfTest`/`IfTrueExpr`/`IfTrueBlock`/`IfMaybeElse`/`IfElse`/`IfElseExpr`/`IfElseBlock` | `if/elif/else` 条件链各阶段 |
| `OkTest`/`OkTestBlock`/`OkTestExpr` | `ok?` 操作符的短路测试 |
| `TryTest*`/`TryErr*`/`TryOk*` | `try/err/ok` 错误处理表达式 |
| `FnMaybeLet`/`FnLetMaybeArgs`/`FnArgList`/`FnArgMaybeType`/`FnArgType`/`FnArgTypeAssign`/`FnBody`/`FnReturnType`/`FnBodyTyped`/`EndFnBlock`/`EndFnExpr`/`EmitFnArgTyped`/`EmitFnArgDyn` | 函数定义/λ表达式各阶段（参数列表、类型标注、返回类型、主体） |
| `EmitUnary`/`EmitSplat`/`EmitOp`/`EmitFieldAssign`/`EmitIndexAssign` | 操作符发射（二元/一元/展开/字段赋值/索引赋值） |
| `EndBare`/`EndBareSquare`/`EndProto`/`EndProtoInherit`/`EndScopeInherit`/`EndFieldInherit`/`EndIndexInherit`/`EndRound` | 括号/花括号/方括号结 |
| `CallMaybeDo`/`EmitCallFromDo`/`EndCall` | 函数/方法调用（含 `do` 语法糖） |
| `ArrayIndex` | 数组索引 `[...]` |
| `EmitReturn`/`EmitBreak`/`EmitContinue` | 控制流语句 |
| `Use`/`Let`/`LetDynOrTyped`/.../`EmitLetDyn`/`EmitLetTyped` | `let` 声明（含动态/类型化两种） |
| `LetArrayDestruct*`/`LetObjectDestruct*`(含嵌套变体) | 解构赋值模式（数组/对象，支持嵌套） |
| `Var`/`VarDynOrTyped`/.../`EmitVarDyn`/`EmitVarTyped` | `var` 可变声明 |
| `MatchSubject`/`MatchBlock`/`MatchArmPattern`/`MatchArmArrow`/`MatchArmBody`/`MatchArmBlock`/`MatchWildcardArrow`/`MatchWildcardBody`/`MatchWildcardBlock`/`MatchMaybeArm`/`MatchWildcardEnd` | `match` 表达式（被去糖化为 `if/else` 链） |
| `ShortCircuitEnd`/`ShortCircuitAssignEnd` | 短路求值（`&&` / `||` / `|?` / `?=`）末尾补丁 |

### `NestedPattern` 枚举 (行 692–698)

表示解构模式中的**嵌套模式**（当前仅支持一层深度）。用于在 `let [a, {x, y}] = arr` 这类语法中记录嵌套绑定。

```rust
pub enum NestedPattern {
    Object(Vec<LiveId>), // {x, y, z}
    Array(Vec<LiveId>),  // [x, y, z]
}
```

### `ScriptParser` 结构体 (行 700–717)

解析器主结构体，携带所有解析状态和输出。

| 字段 | 说明 |
|------|------|
| `index: u32` | 当前 Token 在 tokenizer 中的位置索引 |
| `opcodes: Vec<ScriptValue>` | 生成的 Opcode 指令序列（输出） |
| `source_map: Vec<Option<u32>>` | 源映射，每 Opcode 对应其源码 Token 索引 |
| `had_error: bool` | 解析过程中是否发生过错误 |
| `state: Vec<State>` | 解析器状态栈（核心） |
| `file: String` | 当前解析的源文件名（用于错误报告） |
| `line_offset/col_offset: (usize, usize)` | 源文件的行列偏移（嵌入代码的场景） |
| `destruct_defaults: Vec<(LiveId, Vec<ScriptValue>, Vec<Option<u32>>)>` | 解构默认值的临时暂存空间：`(绑定ID, Opcode序列, 源映射)` |
| `nested_patterns: Vec<NestedPattern>` | 嵌套模式的暂存列表，索引编码在 ids 列表中 |

### `ParserCheckpoint` 结构体 (行 741–751)

用于**增量解析**的快照。保存自动关闭前的解析器状态，以便新源码到达时可以撤销合成的关闭操作并继续解析。

| 字段 | 说明 |
|------|------|
| `opcodes_len` / `source_map_len` | 自动关闭前 Opcode 序列长度 |
| `token_index` | 自动关闭前的 Token 索引 |
| `state` | 状态栈快照 |
| `destruct_defaults_len` / `nested_patterns_len` | 临时存储长度 |
| `last_opcode` | 自动关闭前最后一条 Opcode（可能被 `set_pop_to_me()` 修改，需恢复） |

---

## 辅助函数

### `error!` 宏 (行 8–12)

调用 `self.report_error()` 报告带位置信息的错误。

---

## `State` 的 impl 块 (行 454–689)

### `is_short_circuit_op(op) -> bool` (行 455–457)
判断操作符是否是短路操作符（`&&`, `||`, `|?`）。

### `short_circuit_opcode(op) -> Opcode` (行 459–466)
返回短路操作符对应的 Opcode：`||`→`LOGIC_AND_TEST`, `||`→`LOGIC_OR_TEST`, `|?`→`NIL_OR_TEST`。

### `operator_order(op) -> usize` (行 494–527)
返回操作符的优先级数值（数字越大，绑定越弱）。例如 `.` 返回 3（最高），`=` 返回 19，复合赋值 `?=` 返回 20（最低）。

### `is_assign_operator(op) -> bool` (行 529–550)
判断是否为赋值类操作符（`=`, `:=`, `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `|=`, `^=`, `<<=`, `>>=`, `?=`, `:`, `<:`, `+:`）。

### `operator_supports_inline_number(op) -> bool` (行 552–570)
判断该操作符是否支持**数字内联优化**：如果右操作数是整数常量，可以将其直接编码在 OpcodeArgs 中而非发射值到栈上。

### `operator_to_field_assign(op) -> ScriptValue` (行 572–589)
将赋值操作符映射到字段赋值 Opcode（如 `=` → `ASSIGN_FIELD`, `+=` → `ASSIGN_FIELD_ADD`）。

### `operator_to_index_assign(op) -> ScriptValue` (行 591–608)
将赋值操作符映射到索引赋值 Opcode（如 `=` → `ASSIGN_INDEX`）。

### `operator_to_unary(op) -> ScriptValue` (行 610–619)
一元操作符映射：`~` → `LOG`, `!` → `NOT`, `-` → `NEG`。

### `operator_to_opcode(op) -> ScriptValue` (行 621–673)
将所有二元操作符映射到对应 Opcode。包含 `===`/`!==`（浅比较）、`?:`/`me.`/`.?` 等特殊形式。

### `is_heq_prio(&self, other: State) -> bool` (行 675–688)
判断当前状态中的操作符优先级是否 **高于等于** 另一状态中的操作符。赋值操作符之间有特殊处理（右结合性）。

---

## `ScriptParser` 核心方法

### `report_error(&mut self, tokenizer, msg)` (行 753–768)
调用 `tokenizer.token_index_to_row_col()` 获取当前位置的行列信息，并通过 `log_with_level` 报告错误。设置 `self.had_error = true`。

### Opcode 辅助方法 (行 770–831)

- **`code_len()`**: 返回当前 Opcode 序列长度 `u32`
- **`code_last()`**: 查看最后一条 Opcode
- **`pop_code()`**: 弹出最后一条 Opcode（含 source_map）
- **`push_code(code, index)`**: 推送一条 Opcode + 源映射
- **`push_code_none(code)`**: 推送一条无源映射的 Opcode（合成指令）
- **`set_pop_to_me()`**: 标记最后一条 Opcode 为 `POP_TO_ME`——表示将其结果放入 `me` 引用（赋值语句的特性，Splash 中语句默认将其结果赋给上下文中的 `me`）
- **`has_pop_to_me()`**: 检查是否已有 `POP_TO_ME`
- **`clear_pop_to_me()`**: 清除 `POP_TO_ME` 标记
- **`set_opcode_args(index, args)`**: 回填 Opcode 参数（用于跳转距离等前向引用）

### `parse_step(&mut self, tokenizer, tok, values) -> u32` (行 835–3960)

**解析器核心——4265 行中占据超过 3100 行的巨大 `match` 语句。** 每个 `State` 变体匹配后，根据当前 Token 类型执行相应的解析动作。

返回 `1` 表示消耗了当前 Token，返回 `0` 表示未消耗（让上层状态继续处理）。

以下按状态类别分节说明：

#### BeginExpr (行 3139–3439)
表达式开始状态。**这是解析器中最繁忙的状态之一**，处理所有表达式起始的情况：
- **Rust 值内联**: 如果 Token 有对应的 Rust 值索引（`tok.as_rust_value()`），直接推入
- **花括号 `{`**: 检查栈中是否有挂起的 `+:` / `+=` 继承操作符，分别派发到 `EndProtoInherit` / `EndScopeInherit` / `EndFieldInherit` / `EndIndexInherit`；否则为裸对象 `BeginBare`
- **方括号 `[`**: 数组字面量 `BEGIN_ARRAY`
- **圆括号 `(`**: 分组表达式 `EndRound`
- **数值/字符串/颜色/布尔/标识符**: 直接作为字面量推入
- **关键字**: `if` `try` `ok` `for` `loop` `while` `match` `use` `fn` `let` `var` `return` `break` `continue` `true` `false` `me` `scope` `nil` → 各自由对应的状态处理
- **一元操作符**: `-` `+` `!` `~` → EmitUnary
- **`@`**: EscapedId
- **`||` / `|`**: λ 表达式的参数列表
- **`. `**: 隐式 `me.` 字段访问
- **`..`**: Splat 展开操作符

#### EndExpr (行 3441–3854)
表达式结束状态。决定表达式之后的操作符/后续动作：
- **一元后置 `?`**: 转换为 `RETURN_IF_ERR`
- **命名操作符**: `is` → `===`, `and` → `&&`, `or` → `||`
- **短路操作符 `&&` `||` `|?`**: 
  1. 先弹出所有等高优先级 EmitOp
  2. 发射 TEST Opcode 并记录 test_slot
  3. 推入 `ShortCircuitEnd` + `BeginExpr` 状态等待右操作数
- **`?=` 懒赋值**: 对简单变量发射 `ASSIGN_IFNIL` + `ShortCircuitAssignEnd`
- **二元操作符**（优先级算法）:
  1. 弹出并发射所有优先级 ≥ 当前操作符的 EmitOp
  2. 检查是否是 `.` / `.?` 链上的赋值（→ EmitFieldAssign）
  3. 用 `is_heq_prio` 决定新操作符的入栈位置（实现左结合/右结合）
  4. 若未消耗则推入 `EmitOp` + `BeginExpr`
- **花括号 `{`**: 
  - 检查是否在 `.` / `.?` / `+:` / `+=` 等操作符后面
  - 否则视为原型字面量 `END_PROTO`
- **圆括号 `(`**: 函数调用 `CALL_ARGS` 或方法调用 `METHOD_CALL_ARGS`
- **方括号 `[`**: 数组索引 `ARRAY_INDEX`

#### BeginStmt (行 3855–3898)
语句开始状态：
- 遇到 `;` / `,` → 标记 `last_was_sep = true` 并跳过
- 遇到闭合括号 `)` `}` `]` → 根据 `last_was_sep` 标记上游状态的 `last_was_sep`，然后 `return 0` 让闭合状态处理
- 否则 → 进入表达式语句：推入 `EndStmt` + `BeginExpr{required:false}`

#### EndStmt (行 3899–3957)
语句结束状态：
- 自保护：如果 `last == current index`，说明解析陷入循环，报错并跳过 Token
- 对 `FOR_END` / `ASSIGN_ME` / `ASSIGN_ME_VEC` / `BREAK` / `CONTINUE` / `ME_SPLAT` / `let` 类 Opcode → 直接回到 BeginStmt（不需要 POP_TO_ME）
- 其他情况 → 调用 `set_pop_to_me()` 将表达式的值赋给 `me`

#### 循环: For/Loop/While (行 845–937)

**`ForIdent`**: 收集 `for x` / `for x, y` / `for x, y, z` 中的迭代变量名（最多 3 个）。遇到 `in` 关键字后进入 `ForBody`。支持可选括号 `for (x, y) in z`。

**`ForBody`**: 发射 `FOR_1`/`FOR_2`/`FOR_3` 指令，然后分析循环体是块还是单表达式。

**`ForExpr`/`ForBlock`**: 循环体结束后发射 `FOR_END`，回填 `FOR_[N]` 指令的跳转参数。

**`Loop`/`While`**: 类似，发射 `LOOP`+`BREAKIFNOT` 序列。`WhileTest` 在条件表达式后发射 `BREAKIFNOT`。

#### 条件: If/Elif/Else (行 3005–3138)

**`IfTest`**: 条件表达式解析后发射 `IF_TEST` opcode。

**`IfTrueExpr`/`IfTrueBlock`**: 真分支结束 → `IfMaybeElse`。

**`IfMaybeElse`**: 检查后续关键字：
- `elif` → 发射 `IF_ELSE`, 回填 `IF_TEST` 跳转, 推入新 `IfTest`
- `else` → 发射 `IF_ELSE`, 记录 else_start
- 否则 → 用 `need_nil` 标记 `IF_TEST`（没有 else 分支时推 nil）

**`IfElseExpr`/`IfElseBlock`**: else 分支结束 → 回填 `IF_ELSE` 跳转。

#### 错误处理: Try/Err/Ok (行 2820–3004)

**`TryTest`**: 发射 `TRY_TEST`，解析主体。
**`TryTestExpr`/`TryTestBlock`**: 回填跳转，进入 `TryErrBlockOrExpr`。
**`TryErrBlockOrExpr`**: 发射 `TRY_ERR`，检查 err 分支中的 `ok(...)` 返回标识。
**`TryOkBlockOrExpr`**: 发射 `TRY_OK`。

#### `ok?` 操作符: OkTest (行 2837–2881)

简化版 try：只检查 `ok?`，没有 err 分支。发射 `OK_TEST` + `OK_END`。

#### 函数定义: Fn (行 2385–2626)

**`FnMaybeLet`**: `fn` 后的标识符 → `FN_LET_ARGS`（带名称的函数定义）。
- 若直接跟 `{` → 零参数函数
- 若跟 `(` → 进入 `FnArgList`

**`FnArgList`**: 解析参数列表（支持尾随逗号）。λ 使用 `|x, y|` 语法。

**`FnArgMaybeType`/`FnArgType`/`FnArgTypeAssign`**: 处理可选的 `:` 类型标注和 `=` 默认值。

**`FnBody`/`FnBodyTyped`**: 检查 `->` 返回类型标注。发射 `FN_BODY_DYN` 或 `FN_BODY_TYPED`。

**`EndFnBlock`/`EndFnExpr`**: 函数体结束 → 发射 `RETURN`，回填函数 Opcode 跳转长度。

#### 声明: Let / Var (行 1245–1369, 2309–2368)

**`Let`**: 支持三种模式：
- `let mut` → 委托给 `Var`
- `let [...]` → 数组解构 `LetArrayDestruct`
- `let {...}` → 对象解构 `LetObjectDestruct`
- `let ident` → 简单 `LetDynOrTyped`

**`LetDynOrTyped`/`LetType`**: 根据 `=` 还是 `:` 决定动态类型还是类型化声明。无初始化值时用 `LET_DYN`。

**`Var`**: 同 Let 但无解构模式，始终为可变绑定。

#### 解构赋值: LetArrayDestruct / LetObjectDestruct (行 1328–2307)

**布局策略**: 解析期间在 opcodes 中暂存 `[ids..., rhs]`，在 `EmitLetArrayDestruct`/`EmitLetObjectDestruct` 阶段重排为：
```
[rhs, id_0, EXTRACT(0), id_1, EXTRACT(1), ..., DROP, defaults_ifnil...]
```

**嵌套模式支持**: `let [{x, y}] = arr` 和 `let [[x, y]] = arr`。使用 `NOP` 占位符编码嵌套模式索引（非零 args），发射时展开为 `DUP` + 索引 + `ARRAY_INDEX_NIL` + 嵌套提取。

**默认值暂存**: `= default` 的表达式先暂存到 `destruct_defaults`，最后发射为 `[id, ASSIGN_IFNIL(jump), value, ASSIGN]` 实现懒求值。

#### Match 去糖化 (行 938–1215)

`match` 被完全去糖化为 `if/else` 链：
```
match x { 1 => a, 2 => {b} }
→ let $match_N = x
  if $match_N == 1 a else if $match_N == 2 {b}
```

**关键阶段**:
1. `MatchSubject`: 生成临时变量名 `match_{code_len}`，发射 `LET_DYN`
2. `MatchBlock`: 等待 `{`
3. `MatchArmPattern`: 判断是否通配符 `_`，否则解析条件表达式
4. `MatchArmArrow`: 检查 `=>`，发射 `EQ` + `IF_TEST`
5. `MatchArmBody`/`MatchArmBlock`: 处理 arm 体（块或表达式）
6. `MatchMaybeArm`: 回填跳转，检查是否还有更多 arm
7. 通配符臂 `_` 作为最后 else 分支

#### 方法/函数调用 (行 2774–2818)

- `.` 后的 `(` → 方法调用 `METHOD_CALL_ARGS`
- 标识符后的 `(` → 函数调用 `CALL_ARGS`
- `EndCall`: 等待 `)`，进入 `CallMaybeDo`
- `CallMaybeDo`: 如果下一个关键字是 `do` → 特殊 `do` 语法糖（将后面的表达式作为唯一参数传递）

#### 短路求值 (行 2660–2669)

- `ShortCircuitEnd`: 回填 TEST Opcode 的跳转距离（跳过第二个操作数）
- `ShortCircuitAssignEnd`: 先发射 `ASSIGN`，再回填跳转

#### 操作符发射 (行 2628–2659)

- `EmitOp`: 将运算符转为 Opcode 并发射。支持数字内联优化：如果右操作数是整数常量且 `operator_supports_inline_number`，将常量编码在 OpcodeArgs 中。
- `EmitUnary`: 发射一元操作符
- `EmitSplat`: 发射 `ME_SPLAT`
- `EmitFieldAssign`/`EmitIndexAssign`: 字段/索引赋值（含复合赋值）

#### 原型/对象字面量 (行 2705–2773)

六种花括号闭合状态：
| 状态 | Opcode 序列 |
|------|------------|
| `EndBare` | `END_BARE` |
| `EndProto` | `END_PROTO` |
| `EndProtoInherit` | `END_PROTO` + `PROTO_INHERIT_WRITE` |
| `EndScopeInherit` | `END_PROTO` + `SCOPE_INHERIT_WRITE` |
| `EndFieldInherit` | `END_PROTO` + `FIELD_INHERIT_WRITE` |
| `EndIndexInherit` | `END_PROTO` + `INDEX_INHERIT_WRITE` |

### `parse(&mut self, tokenizer, file, offsets, values)` (行 3962–4069)

**主解析入口**。流程：
1. 设置文件名和行列偏移
2. 循环读取 Token，调用 `parse_step` 直到用完所有 Token 或状态栈为空
3. 步进计数保护：如果连续 1000 次 step=0 且状态栈 ≤1，判定死循环并退出
4. **自动关闭阶段**：当 Token 流提前结束时，弹出并处理所有残留状态（闭合未完成的对象/函数/if等）
5. **结尾 RETURN**：检查最后一条 Opcode，若没有 `POP_TO_ME` 也没有 `RETURN`，追加带 `NIL` 的 `RETURN`。脚本的最外层返回值由此确定。

### `save_checkpoint() -> ParserCheckpoint` (行 4074–4084)

保存当前解析器状态的快照。用于增量解析前记录现场。

### `restore_checkpoint(cp: ParserCheckpoint)` (行 4088–4101)

从快照恢复解析器状态。撤销自动关闭阶段产生的合成 Opcode。**关键**：恢复最后一条 Opcode（自动关闭的 `set_pop_to_me` 可能已修改）。

### `parse_streaming(...) -> ParserCheckpoint` (行 4110–4183)

**增量解析入口**。与 `parse()` 的不同：
1. 遇到 `StringUnfinished` Token 时：保存检查点→用提供的完整字符串替换→继续解析剩余 Token→调用 `auto_close`→返回检查点（保留在 `StringUnfinished` 前的位置，下次更多源码到达时可继续）
2. 正常结束时：先保存检查点，再调用 `auto_close`，返回检查点

### `auto_close(checkpoint, max_token_index) -> ParserCheckpoint` (行 4189–4256)

执行自动关闭并追加 RETURN Opcode：
1. 弹出并处理所有残留状态（同 `parse()` 的自动关闭阶段）
2. 处理 `POP_TO_ME` 或追加 RETURN
3. 返回传入的检查点（标记"这是自动关闭前的状态"）

### `dump_opcodes()` (行 4258–4264)

调试辅助方法，打印所有已发射的 Opcode 序列。

---

## 关键架构特性

1. **无 AST 架构**: 不生成中间表示树，边解析边发射字节码，极大减少内存分配
2. **状态栈驱动**: 单入口 `parse_step` + 状态栈替代了传统的递归下降函数树
3. **操作符优先级算法**: 操作符的优先级和结合性通过栈内重排序实现（类 Pratt Parser 但更接近经典 Operator-Precedence）
4. **前向引用回填**: 跳转指令（`IF_TEST`, `IF_ELSE`, `FOR` 等）先用占位符发射，后续通过 `set_opcode_args` 回填跳转距离
5. **数值内联优化**: 右操作数为整数常量的二元运算可以直接编码在 OpcodeArgs 中
6. **增量解析**: `parse_streaming` + `ParserCheckpoint` 支持流式/IDE 场景下的增量重解析
7. **继承操作符**: `+:` / `+=` 在对象/数组/字段作用域中被转换为特殊的继承读写 Opcode
8. **解构赋值懒默认值**: 默认值表达式暂存于 `destruct_defaults`，发射为 `ASSIGN_IFNIL` + `value` + `ASSIGN` 序列
