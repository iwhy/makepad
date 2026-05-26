# `string.rs` — 不可变字符串系统：`ScriptRcString` 与字符串操作方法

## 文件位置
`platform/script/src/string.rs` (849 行)

## 核心类型

### `ScriptRcString(Arc<String>)` — 引用计数不可变字符串
```rust
#[derive(Clone, Default, PartialEq, Eq, Hash)]
pub struct ScriptRcString(pub Arc<String>);
```

#### 设计动机与优化策略
这是 Makepad 脚本引擎中所有字符串值的底层表示。选择 `Arc<String>` 而非 `Rc<String>` 的原因是 **线程安全**：脚本引擎支持线程（多个 `ScriptThread` 并发），而字符串可能被多个线程的堆引用。`Arc` 允许多线程间共享而不需要锁。

#### 内联优化策略（Inlining Strategy）
`ScriptRcString` 作为 `Arc<String>` 的包装，其**内联优化不在 Rust 结构体层面**，而是在 `ScriptValue` 的表示层：

- **短字符串（<=7 字节）**：存储在 `ScriptValue` 的 `u64` 的 `data` 字段中，通过 `inline_string` 编码直接嵌入。完全避免堆分配。
- **长字符串**：回退到 `ScriptRcString`（即 `Arc<String>`），通过 `ScriptHeap` 管理生命周期。

这套策略在 `value.rs`/`heap.rs` 中实现，string.rs 自身只关注长字符串的存储和操作。

#### `new(str: String)` — 创建
将 `String` 包装到 `Arc<String>`。分配发生在 `String` 的创建时，`Arc` 本身是轻量级引用。

#### `Borrow<str>` / `Borrow<String>` 实现
允许 `ScriptRcString` 同时作为 `str` 和 `String` 借出。这使得字符串操作时可以零成本地获取 `&str` 切片。

---

### `StringTag(u64)` — 字符串 GC 标签
精简版的 GC 标签：

| 标志 | 用途 |
|------|------|
| `MARK (0x1)` | GC 标记 |
| `STATIC (0x2)` | 静态字符串，GC 跳过 |

注意这里和 `ScriptObjectTag`/`ScriptArrayTag` 不同：`ScriptStringData` 没有冻结标志。字符串天然不可变，不需要冻结机制。

---

### `ScriptStringData`
```rust
pub struct ScriptStringData {
    pub tag: StringTag,
    pub string: ScriptRcString,
}
```

---

## 原型方法 (`add_type_methods`)

### `to_bytes` — 字符串转 U8 数组
委托 `heap.string_to_bytes_array(sself)`：将字符串的字节视图转换为 `ScriptArrayStorage::U8` 类型的数组。

### `to_chars` — 字符串转字符数组
委托 `heap.string_to_chars_array(sself)`：将字符串的每个 `char`（Unicode 标量值）转换为 `ScriptArrayStorage::U32` 数组。

### `len` — 字符串长度
实现逻辑：
1. `heap.string_with(sself, |_heap, s| s.len())` 获取字节长度。
2. 转换为 `(len as f64).into()` 作为数值返回。
3. 注意 Rust 的 `str.len()` 返回字节数而非字符数。

### `to_f64` — 字符串转浮点数
实现逻辑：
1. 调用 `s.parse::<f64>()` 解析字符串为数字。
2. 解析失败则返回 `f64::NAN`，但通过 `ScriptValue::from_f64_traced_nan` 嵌入调用栈 IP 以便调试。
3. 成功则直接 `ScriptValue::from_f64(result)`。

### `parse_json` — JSON 解析
实现逻辑：
1. 从当前线程暂时取出 `json_parser`（避免与 `heap.string_mut_self_with` 的借用冲突）。
2. 通过 `string_mut_self_with` 获取字符串切片。
3. `json_parser.read_json(s, heap)` 执行解析。
4. 归还 `json_parser` 到线程。
5. 这是一种**所有权借用模式**：因为 `json_parser` 是 `ScriptThread` 的字段，而 `heap` 需要可变引用，必须将 parser 临时移出线程以避免同时借用。

### `trim` — 去除首尾空白
实现逻辑：
1. 调用 `sself.trim()`（Rust 标准库方法）。
2. `heap.new_string_from_str(trimmed)` 在堆上创建新字符串。
3. 返回新字符串。原有字符串不受影响。

### `strip_prefix(pat)` — 去除前缀
实现逻辑：
1. 嵌套 `string_mut_self_with` 获取自身和模式字符串。
2. 调用 `sself.strip_prefix(pat)`。
3. 如果匹配成功，返回去掉前缀后的新字符串；否则返回原字符串的副本。

### `strip_suffix(pat)` — 去除后缀
与 `strip_prefix` 逻辑相同，但使用 `strip_suffix` 方法。

### `split(pat)` — 分割
支持两种模式：

**字符串模式**：
1. 创建新数组 `heap.new_array()`。
2. 确保存储类型为 `ScriptValue`。
3. 遍历 `sself.split(pat)` 迭代器，每个片段通过 `heap.new_string_from_str` 创建新字符串后追加到数组。

**正则表达式模式**（`pat.as_regex()`）：
1. 调用 `regex_split(heap, sself, re_ptr)`。
2. 正则分割：找到所有非重叠匹配，将匹配之间的部分作为分割片段。
3. 同时包含捕获组到结果中（符合 JavaScript `String.prototype.split` 行为）。

### `search(pat)` — 搜索子串
返回第一个匹配的**字节索引**，未找到返回 `-1.0`。

**字符串模式**：
1. 调用 `sself.find(pat)`。
2. 返回索引或 `-1.0`。

**正则模式**：
1. 创建 2 个 `Option` 槽位。
2. `re.inner.run(s, &mut slots)` 执行匹配。
3. `slots[0].unwrap_or(0)` 作为匹配起始位置返回。

### `match_str(pat)` — 正则匹配

**字符串模式**（简单 indexOf 风格）：
1. 调用 `sself.find(pat)`。
2. 如果找到，构造一个对象 `{value: pat, index: idx}`。
3. 未找到返回 NIL。

**正则非全局模式**：
1. 调用 `regex_exec_first(heap, s, re_ptr, num_captures)`。
2. 返回 `{value: match_str, index: start_pos, captures: [full_match, cap1, cap2, ...]}`。
3. 捕获组数组以 `[0]` 为完整匹配，`[1..n]` 为各分组。

**正则全局模式**：
1. 调用 `regex_match_all_strings(heap, s, re_ptr)`。
2. 返回只包含匹配字符串的数组：`["match1", "match2", ...]`。

### `match_all(pat)` — 全部匹配详细信息
`pat` 必须是正则表达式。
1. 调用 `regex_match_all_detail(heap, s, re_ptr, num_captures)`。
2. 返回 `[{value, index, captures}, ...]` 数组，每个元素包含完整的匹配详情。
3. `captures` 数组格式同 `match_str`。

### `replace(pat, rep)` — 替换

**字符串模式**：
1. 调用 `sself.replacen(pat, rep, 1)`，仅替换第一个匹配。

**正则模式**：
1. 获取替换字符串 `rep_str`。
2. 调用 `regex_replace(heap, s, re_ptr, &rep_str)`。
3. 支持替换引用：`$&`（完整匹配）、`$1-$9`（捕获组）、`$$`（字面量 `$`）。
4. 全局标志控制替换全部还是仅第一个。

### `url_decode` — URL 解码
实现逻辑：
1. 调用 `percent_decode(sself)` 处理 `%XX` 编码。
2. `+` 号转为空格（兼容 `application/x-www-form-urlencoded`）。
3. 非编码字符直接保留。

### `url_encode` — URL 编码
实现逻辑：
1. 遍历字符串的每个字节。
2. 字母数字和 `-_.~` 直接保留（非保留字符，RFC 3986）。
3. 其他字节 `%XX` 格式编码。

---

## 正则辅助函数

### `regex_find_all(heap, input, re_ptr, num_captures)` — 查找所有非重叠匹配
核心实现逻辑：
1. 分配 `(num_captures + 1) * 2` 个槽位（每个捕获组需要起始和结束两个槽位）。
2. 从 `search_from = 0` 开始循环。
3. 每次循环：清除槽位，创建 `haystack = &input[search_from..]` 子串。
4. `re.inner.run(haystack, &mut slots)` 执行匹配。
5. 将结果偏移回原始坐标：`match_start = search_from + slots[0]`。
6. 提取各捕获组范围（同样是绝对坐标）。
7. 推进：如果匹配长度 > 0，`search_from = match_end`；否则（零长度匹配）推进到下一个字符边界，避免死循环。

### `next_char_boundary(s, pos)` — 下一个字符边界
从 `pos + 1` 开始逐个字节检查，直到 `s.is_char_boundary(p)` 为真。这是为了防止在 UTF-8 多字节序列中间截断。

### `regex_split(heap, input, re_ptr)` — 正则分割
实现逻辑：
1. 调用 `regex_find_all` 获取所有匹配范围。
2. 创建数组。
3. 遍历匹配：将 `last_end` 到匹配起始的文本加入数组，然后加入每个捕获组的值。
4. 最后加入 `last_end` 到末尾的剩余文本。
5. 模拟 JavaScript `String.prototype.split` 行为（包含括号捕获组）。

### `regex_match_all_strings(heap, input, re_ptr)` — 全部匹配字符串
对每个匹配范围提取 `input[start..end]` 子串并收集到数组。

### `regex_exec_first(heap, input, re_ptr, num_captures)` — 首次匹配详情
实现逻辑：
1. 准备槽位，执行 `re.inner.run(input, &mut slots)`。
2. 提取匹配范围和值。
3. 创建结果对象，设 `value`、`index` 属性。
4. 创建 `captures` 数组：`[0]` 为完整匹配字符串，`[1..n]` 为各捕获组字符串或 NIL。
5. 将 `captures` 数组设为对象的属性。

### `regex_match_all_detail(heap, input, re_ptr, num_captures)` — 全部匹配详情
对 `regex_find_all` 的每个结果，调用与 `regex_exec_first` 相同的构建逻辑创建详情对象，收集到数组。

### `regex_replace(heap, input, re_ptr, replacement)` — 正则替换
实现逻辑：
1. 判断是否全局模式，非全局则只取第一个匹配。
2. 无匹配时返回输入字符串副本。
3. 遍历匹配：追加匹配之间的文本，调用 `expand_replacement` 展开替换引用。
4. 最后追加剩余文本。

### `expand_replacement(out, replacement, matched, input, caps)` — 展开替换模式
实现逻辑：
1. 逐字节扫描 `replacement`。
2. 遇到 `$` 时检查下一个字符：
   - `$$` → 字面量 `$`
   - `$&` → 完整匹配文本
   - `$0`-`$9` → 解析最多连续数字作为捕获组号（支持 `$10` 等两位数以上），查找对应捕获组文本
   - 其他 → 保留 `$` 字符
3. 非 `$` 的 UTF-8 字符直接追加。
4. 注意 `$` 后紧跟非数字/`$`/`&` 的处理：只追加 `$` 本身，格式化字符在下次循环处理。

---

## 工具函数

### `percent_decode(input)` — URL 百分号解码
1. 预分配 `String::with_capacity(input.len())`（解码后通常更短）。
2. 遍历输入字节：
   - `%XX` → 解析两个十六进制数字，组合成一个字节。
   - `+` → 空格。
   - 其他 → 直接复制。
3. 直接 `(hi << 4 | lo) as char` 可能产生非法 UTF-8，因为 `%XX` 解码后可能不是有效 UTF-8 序列。这里依赖 Rust 的 `String::push` 在无效码点时的行为。

### `hex_val(b)` — 十六进制字符转数值
将 ASCII 十六进制字符 `0-9`、`a-f`、`A-F` 转为对应的数值 `0-15`。不能识别的字符返回 `None`。

### `percent_encode(input)` — URL 百分号编码
1. 预分配 `input.len() * 3`（最坏情况每个字节都需编码）。
2. 对每个字节判断：
   - 非保留字符（字母数字 `-_.~`）直接保留。
   - 其余字节编码为 `%XX` 大写十六进制。
