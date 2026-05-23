# `value_map.rs` 源码解读

**路径:** `platform/script/src/value_map.rs`
**行数:** 119
**核心职责:** 提供为脚本引擎优化的、基于 `LiveId` 的无哈希 HashMap 包装。使用自定义的 `ValueHasher`（直接将 u64 哈希值作为输出，不做任何计算），适用于 key 已经是预计算哈希值的场景。

---

## 结构体 `ValueHasher`

**路径:** 第9-51行

自定义 `std::hash::Hasher` 实现，仅接受 `u64` 类型的输入（`write_u64`），直接将其作为哈希结果输出。其他 `write_*` 方法均 `unreachable!`。这使得此 Hasher 可以在已知 key 就是最终哈希值的情况下完全跳过计算。

### 方法
- **`write_u64(&mut self, n)`**：将 n 直接存入 `self.0`
- **`finish(&self)`**：返回 `self.0`
- 其他 write 方法：调用即 panic

---

## 结构体 `ValueHasherBuilder`

**路径:** 第53-63行

实现 `std::hash::BuildHasher`，每次返回一个新的 `ValueHasher`。

---

## 结构体 `ValueMap<K, V>`

**路径:** 第65-69行

`HashMap<K, V, ValueHasherBuilder>` 的新类型包装，使用无操作哈希器。

### 约束
K 必须满足 `Eq + Hash + Copy + From<LiveId> + Debug`。

### 默认构造（第71-81行）
使用 `HashMap::with_hasher(ValueHasherBuilder{})` 创建。

### Deref / DerefMut（第83-100行）
透明地委托内部的 HashMap 实现。

### Index / IndexMut（第102-119行）
通过 `HashMap::get` / `get_mut` 提供索引访问，未找到时 unwrap 会 panic（调用者保证 key 存在）。

---

## 设计意图

`ValueMap` 主要用于脚本引擎中 key 是 `LiveId` 的场景。`LiveId` 本身是一个预计算的 u64 哈希值，使用 `ValueHasher` 可以零成本地将其作为 HashMap key，避免重复哈希计算。这在不产生额外哈希开销的前提下获得了 HashMap 的 O(1) 查找性能。
