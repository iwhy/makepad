# `json.rs` 源码解读

**路径:** `platform/script/src/json.rs`
**行数:** 351
**核心职责:** 实现 JSON 解析器，将 token 流解析为 Makepad 脚本堆上的 ScriptValue。基于状态机风格，复用 ScriptTokenizer 的 token 输出。

---

## 枚举 `State`

**路径:** 第8-15行

解析器内部的状态机状态：
- `Root`：根级别，等待第一个 token
- `RootMaybeObject`：根级别已有一个值，期待 `:` 表示该值作为对象 key
- `ObjectKey(obj)`：在对象中解析 key
- `ObjectColon(obj, key)`：已解析 key，期待 `:`
- `ObjectValue(obj, key)`：已遇到 `:`，解析 value
- `Array(arr)`：在数组中解析元素

---

## 结构体 `JsonParser`

**路径:** 第17-33行

### 字段
- `index: u32`：当前处理的 token 位置
- `root: ScriptValue`：解析结果的根值
- `state: Vec<State>`：状态栈（允许嵌套对象和数组）
- `errors: Vec<(u32, String)>`：错误列表，每个条目包含 token 索引和描述

### 方法
- **`clear()`**：重置解析器状态、根值、错误列表。初始状态推入 `State::Root`。

---

## impl JsonParser 核心解析

### `parse_step(&mut self, tok, heap)`（第36-321行）

单一 token 处理步骤，根据当前状态分派：

**`State::Root`**（第63-118行）
- `true/false/null` 标识符 → 赋值根值
- 普通 Identifier → 作为 id 值，push `RootMaybeObject`
- String/U40/F64/Color → 赋值根值，push `RootMaybeObject`
- `OpenCurly` → 创建新对象，设置 string_keys，push `ObjectKey`
- `OpenSquare` → 创建新数组，push `Array`
- `CloseCurly/CloseSquare/CloseRound` → 报错

**`State::RootMaybeObject`**（第38-61行）
- `:` 操作符 → 将之前的值当作 key，进入 `ObjectValue` 状态
- 其他操作符 → 报错

**`State::ObjectKey(obj)`**（第119-174行）
- Identifier/String/F64/Color → 作为 key，push `ObjectColon`
- `OpenCurly/OpenSquare` → 递归创建嵌套对象/数组作为 key
- `CloseCurly` → 对象结束（静默 pop）
- `,` → 继续解析下一个 key
- 其他 → 报错

**`State::ObjectColon(obj, key)`**（第176-199行）
- `:` 操作符 → 进入 `ObjectValue`
- 其他 → 报错

**`State::ObjectValue(obj, key)`**（第200-261行）
- `true/false/null` → `heap.set_value_def`
- Identifier/String/U40/F64/Color → 设置值
- `OpenCurly/OpenSquare` → 创建嵌套结构体
- `CloseCurly` → 报错（前一个 pop 已处理）
- `,` → 下一个 key

**`State::Array(arr)`**（第262-320行）
- Identifier/String/U40/F64/Color → `heap.array_push_unchecked`
- `OpenCurly/OpenSquare` → 创建嵌套结构体
- `CloseSquare` → 数组结束（pop 状态）
- `,` → 下一个元素
- 其他 → 报错

### `parse(&mut self, tokens, heap)`（第324-334行）

主解析入口。从当前 index 开始遍历 token 数组，对每个 token 调用 `parse_step`，直到所有 token 消费完毕或状态栈为空。

---

## 结构体 `JsonParserThread`

**路径:** 第337-351行

组合了 tokenizer 和 parser 的高层级解析器。

### `read_json(&mut self, json, heap)`（第344-350行）

一次性 JSON 解析接口：
1. 清空 tokenizer
2. 清空 parser
3. tokenize 输入字符串
4. parse token 列表
5. 返回 parser.root
