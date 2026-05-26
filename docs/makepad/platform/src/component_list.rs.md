# `component_list.rs` — 基于可见性追踪的有序键值容器

## 用途

`ComponentList<K, V>` 是一个同时维护插入顺序（`Vec<(K, V)>`）与可见性集合（`HashSet<K>`）的容器。它在 Makepad 引擎中用于管理需要按序遍历、但支持按"可见性"批量删除的组件集合——典型的场景是帧间构建的临时绘制/交互对象列表，每帧重新 push 并 `retain_visible` 清理上一帧过期项。

## 结构定义

```rust
pub struct ComponentList<K, V> {
    list: Vec<(K, V)>,
    visible: HashSet<K>,
}
```

- `list`: 顺序存储的键值对向量，保持插入顺序。
- `visible`: 当前"有效"的键集合，标记哪些条目应被保留。

## 方法详解

### `retain_visible(&mut self)`

遍历 `list`，仅保留 `visible` 中存在的键对应的条目，然后清空 `visible`。这是最典型的帧间清理模式：每帧将所有活动组件的键通过 `push` 重新注册到 `visible`，然后调用此方法删除上一帧未出现的旧条目，实现增量的存活管理。

### `retain_visible_and<CB>(&mut self, cb: CB)`

在 `retain_visible` 的基础上增加一个额外的保留条件闭包 `cb`。当某个键不在 `visible` 中但闭包返回 `true`，该条目也会被保留。这用于同时满足"帧可见性"和"其他策略"的场景，例如既要删除不可见组件又要保留某些正在执行动画的组件。

### `push(&mut self, key: K, value: V)`

将键插入 `visible` 集合，同时将键值对追加到 `list` 末尾。注意：同一个键可以被多次 `push` 而不去重（`list` 允许重复键），但 `visible` 集合保证键的唯一存在性，后续 `retain_visible` 会保留所有匹配该键的条目。

### `len(&self) -> usize`

返回 `list` 中当前条目的总数。注意这可能大于 `visible` 大小，因为 `visible` 只记录键的存在性而不记录重复次数。

### `with_capacity(capacity: usize) -> Self`

预分配 `list` 的容量以减少动态扩容，`visible` 保持默认空状态。适合在已知大致条目数的高频场景使用。

### `extend(&mut self, iter: impl Iterator<Item = (K, V)>)`

对迭代器中的每个元素依次调用 `push`，批量填充数据。迭代器的惰性特性允许它用于链式生成。

### `pop(&mut self) -> Option<(K, V)>`

弹出 `list` 末尾的键值对，并从 `visible` 中移除对应的键。注意：如果同一个键在 `list` 中有多个副本，`visible` 移除后其他副本在下次 `retain_visible` 时会被删除。

### `get(&self, key: usize) -> Option<&(K, V)>`

按位置索引访问 `list` 中的元素，而非按键查找。这与 `Index` trait 的按键查找形成互补。

### `insert(&mut self, index: usize, item: (K, V))`

在指定位置插入条目，同时将键加入 `visible`。此操作可能触发 `Vec::insert` 的 O(n) 元素移动。

## Trait 实现

- **`Index<K>`**: 线性扫描 `list` 找到第一个匹配键并返回其值的不可变引用。若未找到则 panic。
- **`IndexMut<K>`**: 同上，返回可变引用。
- **`Deref<Target=Vec<(K, V)>>`**: 允许 `ComponentList` 透明地使用 `Vec` 的所有方法（`iter`、`sort` 等）。
- **`DerefMut`**: 允许对底层 `Vec` 的可变访问。注意：直接通过 `DerefMut` 修改 `list` 不会同步更新 `visible`。
- **`From<Vec<(K, V)>>`**: 将普通向量转换为 `ComponentList`，所有键均注册为可见。
- **`FromIterator<(K, V)>`**: 允许 `collect` 构建，所有键均注册为可见。

## 设计要点

1. **可见性驱动的生命周期**：容器不提供单独的"删除"API，而是依赖"白名单"模式——每帧重新 `push` 存活对象，再 `retain_visible` 清除过期者。这种模式避免了析构顺序问题和细粒度的增删改操作。
2. **不强制键唯一**：`list` 可以包含重复键，但 `visible` 中的键是唯一的。这在某些需要保留多条同键记录的场景中有用，但大部分使用者应避免重复。
3. **手动同步风险**：通过 `DerefMut` 直接修改 `list` 不会更新 `visible`，可能导致不一致状态。这是有意为之——框架使用者应当在高层 API 上操作而非绕过封装。
