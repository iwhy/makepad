# `mod_html.rs` — HTML 解析与 CSS 选择器模块

## 概述

本文件实现了 Splash 脚本对 HTML 字符串的解析和结构化查询能力。它依赖 `makepad_html` crate 完成原始解析，并通过一套自定义的零拷贝 CSS 选择器引擎在节点树上执行查询。模块注册为 `mod.html` 类型（通过 `native.new_handle_type`），提供 `parse_html`、`query`、`attr`、`array` 和属性 getter（`length`、`text`、`html`）。

---

## 核心数据结构

### `HtmlBacking`

```rust
struct HtmlBacking {
    decoded: String,
    nodes: Vec<HtmlNode>,
}
```

解析后的 HTML 负载。`decoded` 是解码后的原始字符串，`nodes` 是解析出的扁平节点序列。整个结构被 `Rc` 包裹，使得多次查询（`query`）和数组拆分（`array`）操作可以共享同一份数据而不复制。

### `ScriptHtmlDoc`

```rust
pub struct ScriptHtmlDoc {
    backing: Rc<HtmlBacking>,
    ranges: Vec<(u32, u32)>,  // (open_tag_index, close_tag_index) 对
    is_root: bool,
    handle: ScriptHandle,
}
```

脚本可见的 HTML 文档句柄。`ranges` 存储当前查询命中的元素范围（`(起始 OpenTag 索引, 对应 CloseTag 索引)`）；`is_root` 为 `true` 且 `ranges` 为空时表示整个根文档。结构实现了 `ScriptHandleGc`，支持 GC 追踪。

---

## CSS 选择器解析（`parse_query`）

### 词法单元类型

- **`Combinator`**: `Descendant`（后代选择器，空格）或 `Child`（子选择器，`>`）。
- **`Terminal`**: 查询的终结模式：
  - `None`: 返回匹配元素的句柄数组
  - `Attr(LiveId)`: 返回元素的指定属性值（`@attr`）
  - `Text`: 返回元素内的文本内容（`.text`）
  - `Index(usize)`: 按索引取单个匹配元素（`[N]`）
- **`QueryStep`**: 单个选择器步骤，包含 `tag`、`id_filter`（`#id`）、`class_filter`（`.class`）和组合子。

### `parse_query(sel: &str) -> ParsedQuery`

**实现逻辑**:
1. 去掉输入字符串首尾空白。
2. 逐字符扫描，跳过连续空格。遇到 `>` 则将下一个组合子设为 `Combinator::Child`，然后继续。
3. 否则从当前位置读取 token（直到空格或 `>`）。
4. 对每个 token 依次解析修饰符：
   - `@attr` → 终端模式设为 `Terminal::Attr`，标签名截断至 `@` 之前。如果前面为空则默认为 `*`。
   - `[N]` → 终端模式设为 `Terminal::Index(N)`，标签名截断至 `[` 之前。
   - `#id` → 取 `#` 之后的部分作为 `id_filter`，标签名截断至 `#` 之前。前面为空则默认 `*`。
   - `.class` 或 `.text` → 若后缀为 `"text"` 则设为 `Terminal::Text`，否则设为 `class_filter`，标签名截断至 `.` 之前。前面为空则默认 `*`。
5. 剩余部分作为 tag 名，`*` 表示 `None`（匹配任意标签）。
6. 将组合子的上一个值赋值给 `step.combinator`（`next_combinator` 默认 `Descendant`，遇到 `>` 时临时切换为 `Child`）。
7. 收集所有步骤到 `ParsedQuery` 返回。

---

## 查询执行引擎

### `find_close_tag(nodes, open_idx) -> usize`

**实现逻辑**:
1. 从 `open_idx + 1` 向后遍历，维护一个 `depth` 计数器。
2. 遇到 `OpenTag` 加深，`CloseTag` 减小。
3. 当 `depth == 0` 且遇到 `CloseTag` 时，返回该索引。
4. 若未找到（畸形 HTML），返回 `nodes.len() - 1`。

### `element_matches(decoded, nodes, idx, step) -> bool`

**实现逻辑**:
1. 检查 `nodes[idx]` 是否为 `OpenTag`，提取其标签 `lc`。
2. 如果 `step.tag` 存在，校验标签是否匹配。
3. 如果既无 `id_filter` 也无 `class_filter`，直接返回 `true`。
4. 否则从 `idx + 1` 开始遍历属性节点（`Attribute`），直到遇到非属性节点：
   - 在 `id` 属性中匹配 `id_filter`（完整值比较）。
   - 在 `class` 属性中以空格分词后逐项匹配 `class_filter`（LiveId 比较）。
5. 返回 `id_ok && class_ok`。

### `find_elements(decoded, nodes, step, range_start, range_end, recurse, out)`

**实现逻辑**:
1. 在 `[range_start, range_end)` 范围内遍历节点，维护 `depth`。
2. 对每个 `OpenTag` 调用 `element_matches` 检查匹配。
3. 匹配条件为 `(recurse || depth == 0)`——`recurse` 为 true 时扫描所有深度（后代），为 false 时只扫直接子节点。
4. 匹配的元素调用 `find_close_tag` 获取范围，将 `(open_idx, close_idx)` 加入输出。
5. 非递归模式下，跳过该元素内部所有的节点（`i = close + 1`）。

### `execute_query(decoded, nodes, ranges, steps) -> Vec<(u32, u32)>`

**实现逻辑**:
1. 如果 `steps` 为空，直接返回输入 `ranges`。
2. 第一步在输入范围内（若 `ranges` 为空则对全文档）递归搜索匹配元素。
3. 后续步骤对前一步结果逐条搜索：
   - `Child` 组合子：只在直接子节点层搜索（`recurse = false`）。
   - `Descendant` 组合子：递归搜索所有子元素（`recurse = true`）。
   - 搜索范围从 `prev_start + 1` 到 `prev_end`（在匹配元素内部）。
4. 最后对结果排序并去重（同一元素可能通过多条祖先路径被找到）。

---

## 文本与 HTML 重建

### `collect_text(decoded, nodes, start, end) -> &str`

**实现逻辑**:
1. 统计 `[start, end]` 范围内非全空白 `Text` 节点的数量。
2. 如果恰好有一个，直接从 `decoded` 返回切片引用——零拷贝路径。
3. 否则返回空字符串，由调用者转为 `collect_text_owned` 兜底。

### `collect_text_owned(decoded, nodes, start, end) -> String`

**实现逻辑**:
1. 遍历节点范围内的所有 `Text` 节点，累加字符串内容。
2. 非空白文本之间用单空格分隔，纯空白节点只在开头被保留。
3. 返回新分配的 `String`。

### `get_attr(decoded, nodes, open_idx, attr_id) -> Option<&str>`

**实现逻辑**:
1. 从 `open_idx + 1` 扫描直到遇到非 `Attribute` 节点（`OpenTag`、`CloseTag`、`Text`）。
2. 当属性 `lc == attr_id` 时，返回 `decoded` 中该属性值的切片引用。

### `reconstruct_html(decoded, nodes, start, end) -> String`

**实现逻辑**:
1. 创建一个空 `String`，委托给 `reconstruct_html_into`。
2. 从 `start` 到 `end` 遍历节点：
   - `OpenTag`: 输出 `<` + 标签名 + 遍历后续 `Attribute`（输出 ` 属性名="值"` 模式） + `>`。
   - `CloseTag`: 输出 `</` + 标签名 + `>`。
   - `Text`: 直接从 `decoded` 追加原文。
   - `Attribute`: 跳过（已在 OpenTag 阶段处理）。

### `count_top_level_elements(nodes) -> usize`

**实现逻辑**: 遍历节点，只在 `depth == 0` 时计数 `OpenTag`。用于根文档的 `length` getter。

### `top_level_ranges(nodes) -> Vec<(u32, u32)>`

**实现逻辑**: 遍历节点，对每个顶层 `OpenTag` 调用 `find_close_tag` 获取范围写入数组。

---

## 脚本 API 注册（`define_html_module`）

### `"string".parse_html() → html_handle`

**注册**: `native.add_type_method(heap, ScriptValueType::REDUX_STRING, id!(parse_html), ...)`

**实现逻辑**:
1. 从参数中取出 `self` 字符串。
2. 调用 `parse_html(s, &mut None, InternLiveId::No)` 完成解析。
3. 将返回的 `decoded` 和 `nodes` 装入 `Rc<HtmlBacking>`。
4. 构造 `ScriptHtmlDoc`（`is_root: true`），通过 `heap.new_handle(html_type, ...)` 分配句柄标识。
5. 返回句柄值。

### `html.query(sel) → html_handle | string | array`

**注册**: `native.add_type_method(heap, html_type.to_redux(), id!(query), ...)`

**实现逻辑**:
1. 提取 `self` 句柄和选择器字符串。
2. 调用 `parse_query(&sel_str)` 得到 `ParsedQuery`。
3. 从堆中取出 `ScriptHtmlDoc` 的 `backing` 和 `ranges` 引用（Rc 克隆，零拷贝）。
4. 如果文档非根且之前查询结果为空，直接返回空句柄（短路）。
5. 执行 `execute_query`。
6. 根据 `parsed.terminal` 决定返回值类型：
   - `Attr`: 单匹配返回字符串，多匹配返回数组（每个元素对应属性值或 NIL）。
   - `Text`: 单匹配返回合并字符串，多匹配返回字符串数组。
   - `Index(N)`: 返回第 N 个匹配的局部 `ScriptHtmlDoc`（单元素 `ranges`），越界返回 NIL。
   - `None`: 返回包含所有匹配范围 `ranges` 的新 `ScriptHtmlDoc`。

### `html.attr(name) → string | nil`

**注册**: `native.add_type_method(heap, html_type.to_redux(), id!(attr), ...)`

**实现逻辑**:
1. 提取属性名字符串，转为 `LiveId`。
2. 从当前文档中确定第一个匹配元素的 `OpenTag` 索引：
   - 根文档且 `ranges` 为空：在 `nodes` 中找第一个 `OpenTag`。
   - 有 `ranges`：取第一个范围的 `0` 起始索引。
3. 调用 `get_attr` 获取属性值，返回字符串或 NIL。

### `html.array() → [html_handle, ...]`

**注册**: `native.add_type_method(heap, html_type.to_redux(), id!(array), ...)`

**实现逻辑**:
1. 从当前文档句柄获取 `backing` 和 `ranges`。
2. 若为根文档且 `ranges` 为空，调用 `top_level_ranges` 拆出顶层元素范围。
3. 每个范围创建一个独立 `ScriptHtmlDoc`（单个 `ranges` 元素），通过 `heap.new_handle` 分配新句柄。
4. 将所有句柄推入数组后返回。

### Getters: `html.length`, `html.text`, `html.html`

**注册**: `native.set_type_getter(html_type.to_redux(), ...)`

**实现逻辑**:
- **`length`**: 非根或有 `ranges` 时返回 `ranges.len()`；根文档返回 `count_top_level_elements`。
- **`text`**: 对根文档全量调用 `collect_text_owned`；对有 `ranges` 的逐范围收集后用空格连接。
- **`html`**: 对根文档全量调用 `reconstruct_html`；对有 `ranges` 的逐个重建后连接。

为了避免生命周期冲突，所有字符串提取操作在 `handle_ref` 借用内完成字符串构造，释放借用后再通过 `new_string_from_str` 分配为脚本字符串值返回。

---

## 设计要点

- **零拷贝查询引擎**: `Rc<HtmlBacking>` 被所有查询结果共享，`ranges` 仅记录索引对，不复制任何 HTML 数据。
- **延迟文本提取**: `collect_text` 优先使用切片引用（零分配），只有在多文本节点或首节点为空时才回退到 `collect_text_owned`。
- **终端模式统一**: `query()` 的四种终端模式（Attr、Text、Index、None）覆盖了最常见的查询场景，避免了链式调用查询后再手动取值。
- **去重保护**: 多步查询中同一个元素可能通过不同祖先路径被匹配多次，最终执行 `sort_unstable()` 和 `dedup()` 消除重复。
