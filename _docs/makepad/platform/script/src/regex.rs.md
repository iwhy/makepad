# `regex.rs` — 正则表达式包装系统：`ScriptRegexData` 与正则表达式缓存

## 文件位置
`platform/script/src/regex.rs` (150 行)

## 核心类型

### `RegexTag(u64)` — 正则表达式 GC 标签
与 `StringTag` 采用相同的精简标签模式：

| 标志 | 用途 |
|------|------|
| `MARK (0x1)` | GC 标记-清除标记 |
| `STATIC (0x2)` | 静态正则表达式，GC 跳过 |

---

### `RegexFlags` — 正则表达式标志
```rust
pub struct RegexFlags {
    pub global: bool,       // 'g' — 全局匹配
    pub ignore_case: bool,  // 'i' — 忽略大小写
    pub multiline: bool,    // 'm' — 多行模式
    pub dot_all: bool,      // 's' — . 匹配换行符
}
```

这四个标志对应 JavaScript 正则表达式的标准标志集。

#### `parse(flags: &str)` — 从字符串解析标志
实现逻辑：
1. 遍历输入字符串的每个字符。
2. 对每个字符匹配 `g`/`i`/`m`/`s`。
3. **重复标志检查**：如果某个标志已为 `true`，返回 `Err("duplicate flag")`。这符合严格标志解析的预期。
4. **未知标志检查**：不匹配任何已知标志时返回 `Err("unknown regex flag")`。
5. 返回 `RegexFlags`。

#### `to_parse_options()` — 转为内部解析选项
将 `RegexFlags` 映射为底层正则引擎的 `ParseOptions`：
```rust
ParseOptions {
    dot_all: self.dot_all,
    ignore_case: self.ignore_case,
    multiline: self.multiline,
}
```
注意 `global` 标志不在底层解析选项中——它只影响脚本层的行为（`match_str` 需要区分全局/非全局），不影响正则引擎的匹配行为。

---

### `RegexInternKey` — 正则表达式缓存键
```rust
pub struct RegexInternKey {
    pub pattern: String,      // 模式字符串
    pub flags: RegexFlags,    // 标志
}
```
用于堆上的正则表达式**驻留表**（intern table）：相同的模式和标志组合只会编译一次，后续重复使用从缓存取。

---

### `ScriptRegexData` — 堆上的正则表达式数据
```rust
pub struct ScriptRegexData {
    pub tag: RegexTag,              // GC 标签
    pub inner: InnerRegex,          // 底层正则引擎（makepad_regex::Regex）
    pub pattern: String,            // 原始模式串
    pub flags: RegexFlags,          // 标志
    pub num_captures: usize,        // 捕获组数量（不含第 0 组）
}
```

#### `new(pattern, flags)` — 构造正则表达式
实现逻辑：
1. 调用 `flags.to_parse_options()` 获取解析选项。
2. 调用 `InnerRegex::new_with_options(pattern, options)` 编译正则。如果编译失败，将底层的 `RegexError` 转为字符串错误。
3. 调用 `count_captures(pattern)` 扫描模式串，计算捕获组数量。
4. 返回完整的 `ScriptRegexData`。

#### `count_captures(pattern)` — 计算捕获组数量
这是一个**手动解析器**，而非使用正则引擎的特性：

实现逻辑：
1. 逐字节扫描模式串。
2. **转义跳过**：遇到 `\` 跳过两个字节（`i += 2`）。
3. **字符类跳过**：遇到 `[` 跳到匹配的 `]`，期间遇到 `\` 同样跳过转义。
4. **真正计数**：遇到 `(` 且后面不是 `?`（非捕获组标记，如 `(?:...)`、`(?=...)`、`(?!)` 等），计数 +1。
5. 返回计数。

这种方法比实际执行编译后的分析更高效，且不需要底层正则引擎暴露捕获组计数接口。注意它不处理 `(?|)`（分支重置）等高级特性，但 `makepad_regex` 可能不支持这些特性。

---

## 设计总结

整个 regex 系统是一个**轻量包装层**，职责包括：

1. **标志解析与验证**: 从脚本层的 `"gims"` 字符串标志解析为结构化的 `RegexFlags`。
2. **内部编译**: 调用 `makepad_regex` 库进行模式编译，封装编译错误。
3. **捕获组计数**: 手动扫描模式字符串计算 `num_captures`，用于后续匹配结果的捕获组提取。
4. **GC 集成**: 通过 `RegexTag` 支持标记-清除 GC。
5. **驻留缓存**: 通过 `RegexInternKey` 实现模式-标志级别的去重。

实际的匹配执行（`run` 方法）在 `InnerRegex` 中，从 `string.rs` 和 `heap.rs` 调用。`regex.rs` 本身不执行匹配逻辑，只负责数据存储和元数据管理。
