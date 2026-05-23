# `intern.rs` — 字符串内联（String Interning）

## 文件定位

该文件实现了一个**全局字符串内联池**（String Intern Pool），通过 `Intern` trait 为任意字符串提供去重机制。相同内容的字符串共享同一份内存分配，以 `Arc<str>` 的形式存储，从而减少重复字符串的内存占用并支持 O(1) 的相等性比较（通过 `Arc` 指针比较）。

## 核心结构

### `Intern` trait
```rust
pub trait Intern {
    fn intern(&self) -> Arc<str>;
}
```

一个简单的 trait，任何实现了该接口的类型都可以将自身"内联"为全局唯一的 `Arc<str>`。

### `impl Intern for str`
`str` 类型上的实现：

1. 通过 `OnceLock<Mutex<Interner>>` 获取全局单例的内联器
2. 锁定 `Mutex`，调用内部 `Interner::intern()` 方法
3. 使用 `poisoned.into_inner()` 处理锁中毒——即使其他线程 panic 导致锁中毒，也继续执行（获取中毒锁的内部值）

### `Interner` 结构
```rust
struct Interner {
    cached_strings: HashSet<Arc<str>>,
}
```

内部的字符串缓存，使用 `HashSet<Arc<str>>` 存储所有已内联的字符串。

### `intern(&mut self, string: &str) -> Arc<str>`

核心方法逻辑：
1. 检查 `HashSet` 中是否已存在相同内容的字符串
2. 若不存在：将 `string` 转换为 `Arc<str>`（新分配）并插入集合
3. 从集合中取出并返回 `Arc<str>` 的 clone（仅增加引用计数）

## 全局单例

```rust
static INTERNER: OnceLock<Mutex<Interner>> = OnceLock::new();
```

- 使用 `OnceLock` 确保惰性初始化——仅在第一次调用 `intern()` 时才创建 `Interner` 实例
- 使用 `Mutex` 保证线程安全——允许多个线程并发调用 `intern()`
- 锁竞争优化：由于内联操作通常涉及低频率调用，且 `HashSet` 操作开销较小，`Mutex` 的竞争压力不大

## 内存与性能特性

| 操作 | 性能 | 说明 |
|------|------|------|
| 首次插入新字符串 | O(n) 哈希 + O(1) 插入 | 涉及堆分配（创建 `Arc<str>`） |
| 重复插入已存在字符串 | O(n) 哈希 + O(1) 查找 | 仅增加引用计数，无分配 |
| 字符串相等性比较 | O(1) | 通过 `Arc::ptr_eq()` 进行指针比较 |
| 内存占用 | 每个唯一内容一份 | 重复内容共享同一份内存 |
| 线程安全 | 是 | 通过 `Mutex` 同步 |

## 与 `substr.rs` 的关系

两者都涉及字符串共享，但侧重点不同：
- **`intern.rs`** 关注**全局去重**——相同内容的字符串合并为同一份内存，适用于标识符、属性名、标签等高重复场景
- **`substr.rs`** 关注**零拷贝切片**——共享同一份内存的不同连续片段，适用于文本分割和解析

## 潜在应用场景

- 代码编辑器中的关键字和标识符缓存
- HTML/XML 标签名和属性名的去重
- 协议消息中的固定字段名缓存
- 同一字体名称/路径在多处引用时的内存优化
