# `error.rs` 源码解读

**路径:** `platform/script/derive/src/error.rs`
**行数:** 46
**核心职责:** 实现 `script_err_gen!` proc-macro，为 Makepad 脚本系统生成标准化的运行时错误宏

---

## 类型定义

无直接类型定义。该文件仅包含一个顶层函数。

## 函数

### `pub fn script_err_gen_impl(input: TokenStream) -> TokenStream`
**第 8 行**

`script_err_gen!` 宏的实现。接收一个标识符作为参数（如 `type_error`），生成一个名为 `script_err_<ident>!` 的 `#[macro_export]` 宏定义。

生成的宏包含两个重载版本：

1. **单参数版本 `($trap:expr)`（第 17-27 行）：** 接收 trap 表达式作为唯一参数。检查 trap 是否为 `ScriptTrap::Inner`（活动状态），如果是则构造 `ScriptValue::<Ident>` 错误值并推入 trap 栈，同时记录 `stringify!($trap)` 上下文、`file!()` 文件名和 `line!()` 行号。如果 trap 为空（`ScriptTrap::Pass`），返回一个带默认 `ScriptIp` 的空错误值。

2. **多参数版本 `($trap:expr, $($arg:tt)*)`（第 28-38 行）：** 接收 trap 和格式化参数。与单参数版本相同，但使用 `format!($($arg)*)` 生成带格式化字符串的错误消息，允许调用者传入类似 `format!` 语法的参数列表。这是最常见的用法，例如 `script_err_type_mismatch!(trap, "expected f64, got {}", actual_type)`。

**实现细节：**
- 使用 `TokenBuilder` 构建宏定义 token 流
- 通过 `$crate` 前缀避免宏引用的路径问题，确保宏在任何 crate 中都能正确解析到 `trap` 模块
- 错误值始终携带 `trap.ip`（指令指针），用于在错误回溯时定位脚本源码位置
- 返回的错误值类型与宏标识符同名，构成统一的错误类型体系
