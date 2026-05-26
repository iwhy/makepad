# `component_map.rs` — 基于可见性追踪的哈希表容器

## 用途

`ComponentMap<K, V>` 是 `ComponentList` 的哈希表变体，使用 `HashMap<K, V>` 替代 `Vec<(K, V)>` 作为底层存储，提供 O(1) 的键查找能力。它同样维护一个 `visible: HashSet<K>` 来追踪帧间有效的条目。适用于需要快速随机访问组件的场景，同时保持帧间自动清理的能力。

## 结构定义

```rust
pub struct ComponentMap<K, V> {
    map: HashMap<K, V>,
    visible: HashSet<K>,
}
```

- `map`: 键到值的映射，提供 O(1) 查找。
- `visible`: 当前有效的键集合。

## 方法详解

### `retain_visible(&mut self)`

遍历 `map`，仅保留 `visible` 中存在的键项，然后清空 `visible`。与 `ComponentList` 的对应方法功能一致，但由于基于 `HashMap::retain` 实现，其时间复杂度为 O(n) 映射大小而非线性扫描。

### `retain_visible_with<CB>(&mut self, cb: CB) where V: Default`

扩展版的 `retain_visible`：当某个条目被判定为不可见（不在 `visible` 中）时，不是直接丢弃，而是将它的值通过 `std::mem::swap` 替换为 `V::default()` 并传给回调闭包 `cb`。这允许调用者在删除前执行资源的清理或状态回收操作，例如解绑 GPU 资源或释放内存。

### `retain_visible_and<CB>(&mut self, cb: CB)`

在 `retain_visible` 的基础上增加额外的保留条件。不在 `visible` 中但闭包 `cb` 返回 `true` 的条目不会被删除。这用于需要组合"可见性"和"其他策略"的场景。

### `get_or_insert<'a, CB>(&'a mut self, cx: &mut Cx, key: K, cb: CB) -> &'a mut V`

核心的惰性初始化方法。若 `key` 已存在于 `map` 中，直接返回其可变引用；否则调用 `cb(cx)` 创建新值并插入后将引用返回。无论哪种情况都会将 `key` 加入 `visible` 以保证该条目在当前帧不会被清理。`cx` 参数允许构造器访问全局上下文。

### `entry(&mut self, key: K) -> Entry<'_, K, V>`

返回 `HashMap` 的标准 `Entry` API，同时将 `key` 标记为可见。这允许调用者使用 `Entry::or_insert_with` 等灵活的模式，同时保持可见性追踪。

## Trait 实现

- **`Deref<Target=HashMap<K, V>>`**: 透明地提供 `HashMap` 的所有方法（`get`、`contains_key`、`iter` 等）。
- **`DerefMut`**: 允许对底层 `HashMap` 的可变访问。与 `ComponentList` 一样，绕过封装直接修改不会同步到 `visible`。
- **`Index<K>`**: 基于 `HashMap::get` 的索引访问，未找到时 panic。
- **`IndexMut<K>`**: 可变索引访问。

## 与 ComponentList 差异对照

| 特性 | `ComponentList` | `ComponentMap` |
| --- | --- | --- |
| 底层结构 | `Vec<(K,V)>` | `HashMap<K,V>` |
| 查找复杂度 | O(n) 线性扫描 | O(1) 哈希查找 |
| 保持插入顺序 | **是** | 否 |
| `retain_visible_with` | 不支持 | 支持（带清理回调） |
| `get_or_insert` | 不支持 | 支持 |
| `entry` API | 不支持 | 支持 |
| 典型场景 | 有序绘制列表 | 组件/资源快速查找 |

## 设计要点

1. **帧间懒清理模式**：`visible` 作为白名单，每帧通过 `get_or_insert` 或 `entry` 自动标记活跃条目，帧末通过 `retain_visible` 清理过期项。这种模式避免了在单个帧内频繁删除/插入。
2. **`retain_visible_with` 的资源安全**：在删除前通过 `swap` 取出旧值传给回调，确保资源类型的 `Drop` 在可控上下文中运行，而非在 `HashMap::retain` 的内部执行。
3. **`get_or_insert` 的原子性**：将"检查存在性-标记可见-惰性构造"合并为一个原子操作，避免竞态条件和重复构造。
