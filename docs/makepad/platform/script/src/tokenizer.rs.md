# `tokenizer.rs` — Makepad 脚本流式分词器

## 文件概述

`tokenizer.rs` 实现了 Makepad Script 的流式（增量）分词器 `ScriptTokenizer`。它接受不断流入的字符流，逐步将输入切分为 `ScriptToken` 序列。核心特点是**增量处理**：每次调用 `tokenize()` 传入新字符时，分词器会保持内部状态状态机并复用已有 token 列表，从而实现编辑器级别的增量解析。

---

## ScriptToken 枚举 — Token 类型定义

`ScriptToken` 定义了分词器输出的所有词法单元变体：

| 变体 | 含义 | 含有的数据 |
|------|------|-----------|
| `End` | 文件结束标记 | — |
| `StreamEnd` | 流结束标记 | — |
| `Identifier(LiveId)` | 标识符（变量名、关键字等） | `LiveId` 哈希值 |
| `Operator(LiveId)` | 操作符 | `LiveId` 哈希值 |
| `Separator(LiveId)` | 分隔符（逗号、分号） | `LiveId` 哈希值 |
| `OpenCurly` / `CloseCurly` | `{` / `}` | — |
| `OpenRound` / `CloseRound` | `(` / `)` | — |
| `OpenSquare` / `CloseSquare` | `[` / `]` | — |
| `StringUnfinished` | 未完成的字符串（正在增量输入中） | 未完成字符暂存于 `unfinished` 缓冲区 |
| `String(ScriptValue)` | 完成的字符串常量 | `ScriptValue`（堆内字符串句柄） |
| `F32(f32)` | 32 位浮点数（后缀 `f`） | `f32` |
| `U32(u32)` | 32 位无符号整数（后缀 `u`） | `u32` |
| `I32(i32)` | 32 位有符号整数（后缀 `i`） | `i32` |
| `F16(f32)` | 16 位浮点数（后缀 `h`） | `f32`（内部以 f32 存储） |
| `F64(f64)` | 64 位浮点数（无后缀） | `f64` |
| `U40(u64)` | 40 位无符号整数（整数不含 `.`/`e` 且不超过 `0xFF_FFFF_FFFF`） | `u64` |
| `Color(u32)` | 颜色字面量（`#rrggbbaa`） | `u32`（RGBA 打包） |
| `RustValue(u32)` | Rust 值上下文标记 `@(N)` | `u32`（数字 N） |

### ScriptToken 的辅助方法

- `identifier()` / `operator()` / `separator()` — 提取对应变体的 `LiveId`，不匹配时返回空 id
- `f64()` — 提取 `F64` 数值，不匹配时返回 `0.0`
- `as_f64()` / `as_f32()` / `as_u32()` / `as_i32()` / `as_f16()` / `as_u40()` / `as_color()` — 各自变体的 `Option<T>` 安全提取
- `as_string()` — 提取字符串值；对 `StringUnfinished` 返回 `EMPTY_STRING`
- `as_rust_value()` — 提取 `RustValue` 的数字
- `is_identifier()` / `is_operator()` / `is_open_curly()` / `is_close_curly()` / ... — 各类别的布尔判断

---

## ScriptTokenPos — 带位置的 Token

```rust
pub struct ScriptTokenPos {
    pub token: ScriptToken,
    pos: usize,      // 在 original 字符串中的字符偏移
}
```
每个 token 都记录其在原始输入串中的位置，便于报错时定位。

---

## ScriptTokenizer — 核心分词器

### 字段

| 字段 | 类型 | 含义 |
|------|------|------|
| `pos` | `usize` | 当前已处理的字符总数（每次 `tokenize` 累加） |
| `tokens` | `Vec<ScriptTokenPos>` | 已累积的 token 列表 |
| `original` | `String` | 所有已接收的原始字符拼接 |
| `unfinished` | `String` | 未完成字符串的临时缓冲区 |
| `temp` | `String` | 通用临时缓冲区（凑标识符、数字、操作符等） |
| `state` | `State` | 当前状态机状态 |

### 状态机 — `State` 枚举

```
Whitespace → Identifier / Operator / Number / Color / String('"') / String(''')
    → BlockComment / LineComment
    → (各状态结束回退到 Whitespace)

String(bool) → EscapeInString(bool) → AsciiHexInString(bool) / UnicodeHexInString(bool) / UnicodeCurlyInString(bool)
Operator → 可自动检测 `/*` → BlockComment, `//` → LineComment
RustValue → @( 后接数字直到非数字字符
```

- `State::String(bool)` — `true` 表示双引号 `"..."`，`false` 表示单引号 `'...'`
- `State::BlockComment(usize)` — 支持嵌套 `/* /* */ */`，`usize` 为嵌套深度
- `State::MaybeEndBlock(usize)` — 遇到 `*` 后等待 `/` 结束块注释

### 状态转换表

#### Whitespace 状态
```
数字          → Number（将字符推入 temp）
_/$/字母     → Identifier
#            → Color
逗号/分号    → emit_separator(c)
操作符       → Operator
"            → String(true)
'            → String(false)
{ } [ ] ( )  → emit_token_here(对应块标记)
空白         → 忽略
```

#### Identifier 状态
```
_/$/字母数字  → 继续收集
空白          → emit_identifier() → Whitespace
操作符        → emit_identifier() → Operator
逗号/分号     → emit_identifier() → emit_separator() → Whitespace
#             → emit_identifier() → Color
{ } [ ] ( )  → emit_identifier() → emit_token_here() → Whitespace
"             → emit_identifier() → String(true)
'             → emit_identifier() → String(false)
其他          → emit_identifier() → Whitespace
```

#### Operator 状态
最复杂的状态之一。其核心逻辑是：

1. 维护 `temp` 中的当前操作符前缀
2. 见到新字符时尝试扩展（`extended = temp + c`）
3. 通过 `is_valid_operator()` 和 `could_be_operator_prefix()` 判断是否要扩展
4. 特殊规则：`@` + `(` → RustValue 状态
5. `.` + 数字 → Number 状态（处理 `.5` 这种浮点字面量）
6. 操作符完成后自动发射（`is_valid_operator` 且 `!could_be_operator_prefix`）
7. 检测到 `/*` → BlockComment，`//` → LineComment

**`is_valid_operator(s)`** — 检查 `s` 是否为完整合法的操作符字符串。包括：
- 单字符：`!` `~` `+` `-` `*` `/` `%` `&` `|` `^` `<` `>` `=` `.` `?` `:` `@`
- 双字符：`==` `!=` `<=` `>=` `&&` `||` `+=` `-=` `*=` `/=` `%=` `&=` `|=` `^=` `:=` `<<` `>>` `..` `->` `.?` `>:` `<:` `^:` `+:` `?=` `++` `-:` `=>` `/*` `//`
- 三字符：`===` `!==` `<<=` `>>=` `...`

**`could_be_operator_prefix(s)`** — 检查 `s` 是否可能被扩展为更长的操作符。例如 `=` 可以变成 `==` 或 `===`。

#### String / EscapeInString / AsciiHexInString / UnicodeHexInString / UnicodeCurlyInString
- `EscapeInString(double)`：处理 `\n` `\t` `\r` `\0` `\\` `\"` `\'`、`\x`（ASCII hex）、`\u` / `\u{...}`（Unicode hex）
- `AsciiHexInString(double)`：`\xXX` 两位十六进制 → ASCII 字符
- `UnicodeHexInString(double)`：`\uXXXX` 四位十六进制 → Unicode 字符
- `UnicodeCurlyInString(double)`：`\u{...}` 变长十六进制 → Unicode 字符

#### Number 状态
```
数字          → 继续收集
.e/E         → 科学计数法/浮点后缀
+/- 跟在 e/E 后 → 科学计数法指数符号
0x/X         → 十六进制前缀
f            → emit_f32()
u            → emit_u32()
i            → emit_i32()
h            → emit_f16()
_            → 跳过（数字分隔符）
$/$/字母     → emit_f64() → Identifier
#             → emit_f64() → Color
操作符        → emit_f64() → Operator
......       → 处理 `..` 作为范围操作符
```

`emit_f64()` 有一个优化分支：如果数字不含 `.`/`e`/`E` 且值 ≤ `0xFF_FFFF_FFFF`，则发射 `U40(u64)` 而非 `F64(f64)`。

#### Color 状态
- `#(` → RustValue（`#(N)` 语法）
- `0-9a-fA-F` → 收集，满 8 位 → `emit_color()`
- 首字符 `x` → 跳过（`#x` 前缀，用于防止 `#xe2...` 被 Rust tokenizer 误解为科学计数法）
- 字母/操作符/分隔符/块 → `emit_color()` 后切换到对应状态

#### RustValue 状态
- `0-9` → 收集数字
- 非数字 → `emit_rust_value()` → Whitespace

---

## 核心方法

### `clear()`
重置分词器所有状态：`pos` 归零、`tokens`/`original`/`unfinished`/`temp` 清空、`state` 恢复为 `Whitespace`。用于完全重新开始分词。

### `tokenize(&mut self, new_chars: &str, heap: &mut ScriptHeap) -> &[ScriptTokenPos]`
核心增量分词函数。每次调用处理新输入的一段字符：
1. 记录 start 索引（如果上一个 token 是 `StringUnfinished`，则 `start = tokens.len() - 1`，否则 `start = tokens.len()`）
2. 遍历 `new_chars` 中每个字符，将其追加到 `self.original` 并递增 `self.pos`
3. 根据当前 `self.state` 进入对应的状态处理分支
4. 返回本次调用新生成/修改的 token 切片 `&self.tokens[start..]`

这样每次只返回增量部分，上层可以高效地处理编辑器增量输入。

### `emit_identifier()`
将 `temp` 中的字符串通过 `LiveId::from_str_with_lut()` 转换为标识符 `LiveId`。若 LUT 冲突（两个不同字符串哈希到同一 ID），输出警告并改用普通 `from_str`。然后创建 `Identifier` token 并压入 tokens。

### `emit_operator()`
与 `emit_identifier` 类似，但使用 `Operator` token。如果 `temp` 为空则直接返回（无操作）。

### `emit_separator(c)`
发射分隔符 token（逗号或分号）。调用前确保 `temp` 为空（否则 panic），将字符压入 `temp` 后立即发射。

### `emit_f64()`
解析 `temp` 为 `f64`。特殊优化：如果数字不含 `.`/`e`/`E` 且值 ≤ `0xFF_FFFF_FFFF`（即 40 位无符号整数范围），发射 `U40(u64)` 而非 `F64(f64)`，帮助 shader 编译器区分整数和浮点。

### `emit_f32()` / `emit_u32()` / `emit_i32()` / `emit_f16()`
解析 `temp` 为对应的数值类型并发射 token。这些类型由数字后缀 `f`/`u`/`i`/`h` 触发。

### `emit_color()`
调用 `hex_bytes_to_u32()` 将 `temp` 中的十六进制字符串转换为 `u32` 颜色值，发射 `Color(u32)` token。

### `emit_rust_value()`
解析 `temp` 中的十进制数字为 `u32`，发射 `RustValue(number)` token。由 `@(N)` 语法触发。

### `emit_token_here(token)`
在当前位置直接发射一个无附加数据的 token（主要用于块括号 `{}` `[]` `()`）。

### `append_unfinished_string(c)`
将字符追加到 `unfinished` 缓冲区。如果最后一个 token 是 `StringUnfinished` 则复用，否则创建新的 `StringUnfinished` token。

### `finish_string(heap)`
完成字符串收集：弹出最后的 `StringUnfinished` token，将其 `unfinished` 缓冲区内容通过 `heap.new_string_from_str()` 堆分配为 `ScriptValue`，替换为 `String(ScriptValue)` token。若没有 `StringUnfinished`（空字符串 `""`），直接压入 `String(EMPTY_STRING)`。

### `intern_unfinished_string(heap) -> Option<ScriptValue>`
在增量解析边界使用：如果存在 `StringUnfinished` token，将 `unfinished` 缓冲区内容在堆上分配为临时 `ScriptValue` 并返回。**不修改 token 状态**，所以下次 `tokenize()` 可以继续追加。这让解析器在增量编辑时能获取真正的部分字符串内容。

### `iter_strings() -> impl Iterator<Item = ScriptValue>`
遍历所有 `String` token 并 yield 其值。用于 GC（垃圾回收）将 tokenizer 中的字符串标记为根，防止被回收。

### `token_index_to_row_col(tok_index) -> Option<(u32, u32)>`
将 token 索引转换为原始文本中的行列位置。遍历 `original` 字符串从开头到 token 的 `pos` 位置，统计换行符。

### `dump_tokens(heap)`
调试方法：将所有 token 输出到 stdout。根据 token 类型格式化输出：字符串显示引号内的内容、颜色显示十六进制、RustValue 显示 `#(N)` 等。

### `pos_to_loc(pos) -> Option<ScriptLoc>`
将绝对字符位置转换为 `ScriptLoc { row, col }`。在 `original` 中遍历至目标位置，统计经过的换行符。

---

## 颜色十六进制解析 (colorhex.rs)

`hex_bytes_to_u32(buf: &[u8])` 将十六进制字节序列解析为 `u32` 颜色值：

| 长度 | 格式 | 处理方式 |
|------|------|----------|
| 1 | `#w` | 灰度值，扩展到 RGBA（`wwwwwwff`） |
| 2 | `#ww` | 每字节灰度，RGBA（`wwwwwwff`） |
| 3 | `#rgb` | 每位重复一次扩展（`r=r<<4|r`, 类似 `#rrggbb`） |
| 4 | `#rgba` | 每位重复扩展 |
| 6 | `#rrggbb` | 标准 RGB，A=0xff |
| 8 | `#rrggbbaa` | 完整 RGBA |

---

## 设计要点总结

1. **流式增量分词**：每次 `tokenize()` 处理输入片段，维持状态机，返回增量 token 切片
2. **Unfinished String 支持**：编辑器场景下字符串可能跨多次 `tokenize()` 调用，`StringUnfinished` token 配合 `unfinished` 缓冲区实现增量输入
3. **注释嵌套**：`BlockComment(usize)` 支持 `/* /* */ */` 嵌套
4. **类型后缀**：`f`(f32) `u`(u32) `i`(i32) `h`(f16) 后缀实现字面量类型区分
5. **U40 优化**：不含小数点的整数在 f64 范围内发射为 U40，辅助 shader 编译器区分
6. **操作符最长匹配**：通过 `is_valid_operator` + `could_be_operator_prefix` 实现最长操作符匹配（如 `>>=` 优先于 `>>` + `=`）
7. **`#x` 前缀**：解决 Rust tokenizer 将 `#e` 中的 `e` 误认为科学计数法指数的问题
8. **GC 集成**：`iter_strings()` 让 GC 能标记 tokenizer 中的字符串为根引用
