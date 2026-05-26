# `lib.rs` 源码解读（id_macros proc-macro crate 入口）

**路径:** `libs/live_id/id_macros/src/lib.rs`
**行数:** 174
**核心职责:** 定义 `live_id`、`id`、`ids` 等过程宏，在编译期将标识符字符串转换为 `LiveId(u64)` 常量值，避免运行时哈希计算开销。

---

## 类型/结构定义

该文件未定义任何 struct/enum。所有功能通过函数和 proc-macro 入口提供。

---

## 辅助函数

### `fn parse_ident(parser: &mut TokenParser) -> Result<String, TokenStream>`
- **实现逻辑**: 调用 `parser.expect_any_ident()` 从 Token 流中消费并返回下一个标识符。如果当前 Token 不是标识符，则通过 `expect_*` 返回编译错误。用于需要"强制必须存在标识符"的位置。

### `fn eat_ident(parser: &mut TokenParser) -> Option<String>`
- **实现逻辑**: 调用 `parser.eat_any_ident()` 尝试消费一个标识符。如果当前 Token 不是标识符或不匹配，返回 `None` 而非报错。用于"可选标识符"的位置，如 `live_id_num!` 中先尝试吃标识符，失败后再尝试吃标点符号。

### `fn parse_single_token(item: TokenStream) -> String`
- **实现逻辑**: 调用 `TokenStream::to_string()` 将整个 Token 流序列化为字符串，然后移除所有空格。适用于 `live_id!`、`id!` 等接受单一标识符/字符串的宏，确保输入无论写成 `foo` 还是 `"foo"` 都能被统一处理。

---

## Proc-Macro 入口函数

### `#[proc_macro] pub fn live_id(item: TokenStream) -> TokenStream`
- **实现逻辑**: 将输入 Token 流视为单个标识符或字符串，调用 `LiveId::from_str(&v)` 编译期计算哈希值，然后生成 `LiveId (0x..." )` 形式的 Token 流（即 `LiveId(u64)` 构造表达式）。`LiveId` 类型在同 crate 中通过 `use makepad_live_id_macros::*` 被重导出，所以此宏生成的代码可以直接使用构造语法。结果是编译期常量，零运行时开销。

### `#[proc_macro] pub fn some_id(item: TokenStream) -> TokenStream`
- **实现逻辑**: 与 `live_id!` 类似，但生成 `Some(LiveId (0x...))` 的表达式。适用于需要 `Option<LiveId>` 上下文的地方，直接获得 `Some(...)` 包裹的值。

### `#[proc_macro] pub fn id(item: TokenStream) -> TokenStream`
- **实现逻辑**: 接受单个标识符，如果输入字符串非空则生成 `LiveId(哈希值)`，若为空则生成 `LiveId(0)`（即 `empty()`）。当输入为空字符串时，行为与 `LiveId::empty()` 一致。此宏是 Makepad DSL 中最常用的 ID 创建宏。

### `#[proc_macro] pub fn ids(item: TokenStream) -> TokenStream`
- **实现逻辑**: 以 `.` 分隔的标识符链（如 `foo.bar.baz`）为输入。内部嵌套的 `parse` 函数循环遍历 Token 流：
  1. 如果遇到 `{ }` 花括号则将其视为 Rust 代码块，通过 `tb.stream(Some(parser.eat_level()))` 原样插入。
  2. 否则将标识符通过 `LiveId::from_str` 转换为 `LiveId` 常量。
  3. 每个项后插入逗号。
  4. 识别到 Token 流结束时关闭 `]`。
  最终生成 `&[LiveId(0x...), LiveId(0x...), ...]` 的切片引用表达式。

### `#[proc_macro] pub fn ids_array(item: TokenStream) -> TokenStream`
- **实现逻辑**: 委托给 `ids_array_impl`。生成嵌套的切片引用 `&&[&[LiveId(...), ...], ...]` 结构，适用于需要"ID 数组的数组"的场景。

### `#[proc_macro] pub fn ids_list(item: TokenStream) -> TokenStream`
- **实现逻辑**: 完全委托给 `ids_array_impl`，与 `ids_array` 行为相同。两个名字同时存在是为了在不同上下文中提供语义更清晰的命名。

### `ids_array_impl(item: TokenStream) -> TokenStream`（内部函数）
- **实现逻辑**: 解析以 `,` 分隔的组，每组内以 `.` 分隔标识符：
  - 外层循环 `'outer:` 处理用 `,` 分组的多组 ID。
  - 每组内层循环连续解析 `.` 分隔的标识符。
  - 遇到 `,` 时结束当前组（`break`），遇到 EOT 时结束外层循环（`break 'outer`）。
  - 每组生成 `&[LiveId(...), ...]`，外层再包裹 `&[...]`。

### `#[proc_macro] pub fn live_id_num(item: TokenStream) -> TokenStream`
- **实现逻辑**: 接受 `name, number` 格式的输入（如 `foo, 42`）：
  1. 调用 `eat_ident` 吃标识符作为基名。
  2. 必须紧跟 `,`（逗号分隔），否则返回错误 `"please add a number"`。
  3. 用 `parser.eat_level()` 消费剩余 Token 作为数字表达式。
  4. 将基名用 `LiveId::from_str` 哈希得到种子，再使用 `LiveId::from_num(seed, number)` 生成带编号的 ID。
  5. 生成 `LiveId::from_num(<seed_u64>, <number_expr>)` 的调用表达式。

### `#[proc_macro] pub fn id_lut(item: TokenStream) -> TokenStream`
- **实现逻辑**: 先尝试 `eat_ident` 吃标识符，失败则尝试 `eat_any_punct` 吃标点符号。然后将名称以字符串字面量的形式传给 `LiveId::from_str_with_lut(name).unwrap()`。与 `id!` 的区别在于此宏会触发 `LiveIdInterner` 的字符串反查注册，且最终表达式调用 `.unwrap()` 会导致运行时 panic 在哈希碰撞时。

### `#[proc_macro_derive(FromLiveId)] pub fn derive_from_live_id(input: TokenStream) -> TokenStream`
- **实现逻辑**: 委托给 `derive_from_live_id_impl(input)`，后者定义在 `derive_from_live_id.rs` 中。生成 `From<LiveId>`、`From<&[LiveId;1]>` 和 `From<u64>` 三个 trait 的实现。

---

## 辅助模块

### `mod derive_from_live_id`
- **实现逻辑**: 导入同一目录下的子模块，包含 `#[derive(FromLiveId)]` 派生宏的详细生成逻辑。
