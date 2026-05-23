# `regex_heap.rs` — 正则表达式堆操作

## 文件位置
`platform/script/src/regex_heap.rs` (46 行)

## 概述
`regex_heap.rs` 在 `ScriptHeap` 上实现了**正则表达式的创建、驻留和访问**操作。正则表达式通过 `(pattern, flags)` 二元组进行 intern 驻留，避免重复编译同一正则表达式。

---

## 方法详解

### 1. `new_regex(&mut self, pattern: &str, flags_str: &str) -> Result<ScriptValue, String>`
**创建或获取已驻留的正则表达式**。执行流程如下：
1. 使用 `RegexFlags::parse(flags_str)` 解析标志字符串（如 `"i"`、`"m"` 等），如果解析失败则返回 `Err`。
2. 构建 `RegexInternKey { pattern: pattern.to_string(), flags }` 作为驻留键。
3. 在 `regex_intern` 哈希表中查找：如果已存在相同 pattern 和 flags 的正则表达式，直接返回现有索引。
4. 如果未命中驻留表，调用 `ScriptRegexData::new(pattern, flags)` 编译正则表达式。如果编译失败返回 `Err`。
5. 尝试从 `regexes_free` 空闲列表复用槽位——这些槽位已在 GC sweep 阶段被清理并递增了代号。
6. 如果空闲列表为空，将新数据 push 到 `regexes` GenVec（起始代号为 0）。
7. 将新索引插入 `regex_intern` 驻留表。
8. 返回 `ScriptValue` 形式的正则表达式引用。

### 2. `regex(&self, ptr: ScriptRegex) -> Option<&ScriptRegexData>`
**获取正则表达式数据的不可变引用**。通过 `self.regexes[ptr].as_ref()` 从 `Option<ScriptRegexData>` 中取出引用。如果该槽位已被 GC 回收（为 `None`），返回 `None`。

### 3. `regex_mut(&mut self, ptr: ScriptRegex) -> Option<&mut ScriptRegexData>`
**获取正则表达式数据的可变引用**。通过 `self.regexes[ptr].as_mut()` 获取可变引用，用于执行匹配操作（如 `is_match`、`find`、`replace` 等）。
