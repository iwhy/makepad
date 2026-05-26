# `substr.rs` — 零拷贝子字符串

## 文件定位

该文件实现了 `Substr` 类型——一种**零拷贝**的子字符串视图。通过共享底层 `Rc<str>` 引用计数堆分配字符串，`Substr` 可以在不发生内存分配和拷贝的情况下进行切片操作，适用于需要频繁拆分和传递子字符串的文本处理场景（如语法高亮、代码编辑器等）。

## 核心结构

```rust
pub struct Substr {
    parent: Rc<str>,  // 底层共享字符串
    start: usize,     // 子串在 parent 中的起始字节偏移
    end: usize,       // 子串在 parent 中的结束字节偏移（不包含）
}
```

## 主要方法

### `split_at(index) -> (Substr, Substr)`
在指定字节偏移处拆分为两个子串。两个子串共享相同的 `Rc<str>` parent，仅调整各自的 `start` 和 `end` 范围。不发生任何内存分配。

### `parent() -> &Rc<str>`
返回底层引用计数字符串的引用。

### `start_in_parent() / end_in_parent()`
返回子串在 parent 中的起止字节偏移量。这两个值可以用于：
- 比较两个 `Substr` 是否指向同一 parent 的同一范围
- 在外部数据结构中定位子串（如行列表中的偏移计算）

### `as_str() -> &str`
从 parent 中借用当前范围的字符串切片。生命周期受 parent 的 `Rc` 引用计数限制——返回的 `&str` 生命周期等于 `&self` 的生命周期。

### `shallow_eq(other) -> bool`
引用比较——检查两个 `Substr` 是否指向**同一个** `Rc<str>` 的**完全相同**的范围。使用 `Rc::ptr_eq()` 进行指针比较。这是短路优化的关键：`PartialEq` 先尝试 `shallow_eq`，失败才回退到逐字符内容比较。

### `substr(range) -> Substr`
在现有 `Substr` 基础上进行二次切片。将相对范围（相对于当前子串的起止）转换为绝对范围（相对于 parent）：
1. 根据 `RangeBounds` 类型（`..`、`a..b`、`a..`、`..b` 等）计算新的偏移
2. 新 `start = self.start + range_start`，新 `end = self.start + range_end`
3. 使用 `assert!` 确保范围在子串长度内

## Trait 实现

### `Deref<Target = str>`
允许 `Substr` 透明地当作 `&str` 使用，支持所有 `str` 的固有方法（`.len()`、`.chars()`、`.find()` 等）。

### `From` 系列
- **`From<String>`** — `String` → `Substr`，通过 `as_str().into()` 间接调用 `From<&str>`
- **`From<&String>`** — `&String` → `Substr`，同上
- **`From<&str>`** — `&str` → `Substr`，将字符串字面量拷贝到 `Rc<str>` 中（发生一次分配）
- **`From<Rc<str>>`** — `Rc<str>` → `Substr`，直接复用引用计数指针，`start=0, end=len`

### `Eq` / `Hash` / `Ord` / `PartialEq` / `PartialOrd`
- `Hash` 基于 `as_str()` 的内容哈希
- `Ord` / `PartialOrd` 基于字典序比较
- `PartialEq`：优先使用 `shallow_eq` 做引用短路的快速比较，失败后回退到内容相等比较
- `Eq` 标记为完整等价关系

### `Debug`
委托给 `as_str()` 的 `Debug` 实现，输出带引号的字符串字面量格式。

## 内存布局示例

```
内存分配 (Rc<str>):
  "Hello, World!"
   ^             ^
   |             |
   start=0      end=13

substr1 = Substr { parent: Rc, start: 0, end: 5 }   → "Hello"
substr2 = Substr { parent: Rc, start: 7, end: 12 }  → "World"

两个 Substr 共享同一个 Rc<str>，无额外分配。
substr2.substr(1..3) → Substr { start: 8, end: 10 } → "or"
```

## 适用场景

- 语法分析器中的 Token 切片（无需拷贝文本内容）
- 代码编辑器的缓冲区管理（行和字符的零拷贝引用）
- 正则表达式匹配结果的子串提取
- 文本高亮渲染中的分段处理
