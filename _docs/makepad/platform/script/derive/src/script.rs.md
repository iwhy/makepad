# `script.rs` 源码解读

**路径:** `platform/script/derive/src/script.rs`
**行数:** 232
**核心职责:** 实现 `script!`、`script_mod!`、`script_apply_eval!` 三个核心 proc-macro 的代码生成逻辑，负责将 Makepad 脚本 DSL 编译为运行时 `ScriptMod` 结构体

---

## 类型定义

### `struct Lc`（第 118 行，嵌套函数内）
行号列号位置标记结构体，包含 `line: usize` 和 `column: usize` 两个字段。在 `token_parser_to_whitespace_matching_string` 内部函数中定义和使用，用于精确追踪 token 在源码中的行列位置，以便在重构源码字符串时保持精确的空白间距。

### `impl Lc`（第 123 行）
为 `Lc` 提供 `_next_char` 方法，返回列号 +1 的新位置，用于在处理左定界符后跳过该字符位置。

---

## 顶层函数

### `pub fn script_mod_impl(input: TokenStream) -> TokenStream`
**第 9 行**

`script_mod!` 宏的实现。内部调用 `script_impl()` 将输入脚本编译为 `ScriptMod` 表达式，然后将其包装为完整的 `pub fn script_mod(vm: &mut ScriptVm) -> ScriptValue` 函数定义。该函数封装了 `vm.eval(sb)` 调用，通过调用虚拟机求值执行整个脚本模块。这是将脚本模块注册到 Makepad 虚拟机中的标准入口。

### `pub fn script_apply_eval_impl(input: TokenStream) -> TokenStream`
**第 19 行**

`script_apply_eval!` 宏的实现。解析 `(cx, target, { script_code })` 格式的参数：先提取 `cx` 表达式（如 `self` 或 `cx`），再提取 `target` 表达式（如 `self.draw_bg`），剩余部分作为脚本代码块。将脚本代码块前添加 `__script_source__` 标识，通过 `script_impl` 编译为 `ScriptMod` 结构体。输出代码调用 `cx.with_vm(|vm|{...})` 获取虚拟机实例后，在目标上执行 `target.script_apply_eval(vm, script)`，实现在运行时动态求值并将结果应用到 widget 属性上。

### `pub fn script_impl(input: TokenStream) -> TokenStream`
**第 63 行**

`script!` 宏的核心实现。使用 `TokenParser` 解析输入 token 流，调用 `token_parser_to_whitespace_matching_string` 将其转换为精确匹配的源码字符串和插值值列表。生成 `ScriptMod { cargo_manifest_path, module_path, file, line, column, code, values }` 结构体初始化代码。其中 `code` 字段是保留原始空白格式的脚本源码字符串，`values` 字段收集 `#(expr)` 插值表达式并将它们逐个调用 `.script_to_value(vm)` 求值为脚本值。若输入为空（无源码位置），则生成 `ScriptMod::default()`。

### `fn token_parser_to_whitespace_matching_string(parser, span) -> (String, Vec<TokenStream>)`
**第 106 行**

核心 token 到字符串的转换函数。遍历输入 token 流，精确还原与原始源码匹配的空白格式字符串。同时识别 `#(...)` 插值表达式语法：当遇到 `#` 后紧跟左定界符时，将 `#` 从输出中移除，将该表达式的 token 流存入 `values` 列表，并在输出字符串中标记为 `#(index)` 占位符。内部定义了 `delta_whitespace` 函数，用于在 token 之间插入精确数量的空格/换行以匹配原始源码行列位置。

### `fn delim_to_pair(delim: Delimiter) -> (char, char)`
**第 132 行**

将 `proc_macro::Delimiter` 枚举映射为一对起始/结束字符：`Brace -> ('{', '}')`，`Parenthesis -> ('(', ')')`，`Bracket -> ('[', ']')`，`None -> (' ', ' ')`。

### `fn tp_to_str(parser, span, out, values, last_end)`
**第 141 行**

递归的 token 遍历函数，是 `token_parser_to_whitespace_matching_string` 的内部工作函数。处理嵌套的定界符分组（组/括号/方括号）：遇到分组时先写入起始定界符，递归处理分组内容，再写入结束定界符；遇到普通 token 时直接追加其字符串表示。每两个 token 之间调用 `delta_whitespace` 计算并插入正确的空白间距以匹配源码格式。特殊处理 `#(...)` 模式使其不进入输出字符串而作为插值占位符。

### `fn lc_from_start(span: Span) -> Lc`
**第 148 行**

从 `Span::start()` 创建位置标记，获取 token 起始的行号和列号。

### `fn lc_from_end(span: Span) -> Lc`
**第 155 行**

从 `Span::end()` 创建位置标记，获取 token 结束后的行列位置。

### `fn delta_whitespace(now: Lc, needed: Lc, out: &mut String)`
**第 162 行**

为保持源码字符串与原位置匹配而填充空白的关键函数。如果当前位置和目标位置在同一行，仅填充空格到目标列；如果跨行，填充足够的换行使行号匹配，再填充空格到目标列。确保生成的脚本字符串在行号和列号上与输入源码一致，使脚本错误能够指向正确的源码位置。
