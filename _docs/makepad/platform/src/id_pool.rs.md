# `id_pool.rs` — 带代际标记的 ID 池

## 用途

`IdPool<T>` 实现了一个**代际安全的 ID 分配器**，解决了经典的"悬挂指针"问题：当池中某个槽位被释放后，持有旧 ID 的代码仍然可能引用该索引。通过引入代际计数器（generation），每次槽位被重新分配时递增代际，使得旧 ID 无法访问到重新分配后的新数据。

## 核心数据结构

### `IdPoolFree(Rc<RefCell<Vec<usize>>>)`

空闲槽位索引列表，通过 `Rc<RefCell<>>` 实现共享所有权。这意味着 `PoolId` 持有对空闲列表的引用，并在 `Drop` 时自动将 ID 归还池中。

### `IdPool<T>`

```rust
pub struct IdPool<T: Default> {
    pub pool: Vec<IdPoolItem<T>>,
    pub free: IdPoolFree,
}
```

- `pool`: 存储所有已分配和空闲的槽位，按索引访问。
- `free`: 当前可复用的空闲索引列表。

### `IdPoolItem<T>`

```rust
pub struct IdPoolItem<T> {
    pub item: T,
    pub generation: u64,
}
```

每个槽位包含实际数据 `item` 和代际计数 `generation`。实现了 `Deref` 和 `DerefMut`，可以直接访问 `item` 的字段和方法。

### `PoolId`

```rust
pub struct PoolId {
    pub id: usize,
    pub generation: u64,
    pub free: IdPoolFree,
}
```

对外暴露的句柄，包含索引、代际和空闲列表引用。通过代际验证确保访问安全性。

## 方法详解

### `IdPool::slot_count(&self) -> usize`

返回池的总槽位数（包括空闲和活跃）。等于 `pool.len()`。

### `IdPool::free_count(&self) -> usize`

当前空闲槽位的数量。通过 `free.0.borrow().len()` 读取。

### `IdPool::live_count(&self) -> usize`

当前活跃槽位的数量。通过总槽位减去空闲槽位计算（`slot_count() - free_count()`，使用 `saturating_sub` 防止进位错误）。

### `IdPool::alloc(&mut self) -> PoolId`

主分配方法：
1. 从 `free` 栈中弹出一个空闲索引（LIFO 策略）。
2. 若有可用空闲槽，递增其 `generation` 并返回带新代际的 `PoolId`。
3. 若无空闲槽，调用 `alloc_new` 分配全新的槽位。

### `IdPool::alloc_new(&mut self, item: Option<T>) -> PoolId`

在 `pool` 末尾追加一个新槽位：
- 若 `item` 为 `Some` 则使用提供的值，否则使用 `T::default()`。
- 新槽位的 `generation` 从 0 开始。
- 返回的 `PoolId` 代际同样为 0。

### `IdPool::alloc_with_reuse_filter<F>(&mut self, filter: F, item: T) -> (PoolId, Option<T>)`

带条件重用的高级分配方法：
1. 在 `free` 列表中查找第一个满足 `filter` 闭包的槽位。
2. 若找到，从 `free` 中移除该索引，递增代际，将旧值通过 `std::mem::replace` 替换为新值，返回 `(PoolId, Some(旧值))`。
3. 若未找到，调用 `alloc_new` 分配全新槽位，返回 `(PoolId, None)`。

这用于需要"寻找匹配的已释放槽位进行复用"的场景，例如 GPU 资源池中查找相同尺寸的纹理槽位。

### `PoolId::free(&mut self)`

手动释放 ID：将 `self.id` 推入空闲列表。注意，由于 `Drop` 也会自动释放，手动调用可能导致双重释放。通常只在需要提前释放的特定场景使用。

### `Drop for PoolId`

```rust
fn drop(&mut self) {
    self.free()
}
```

RAII 模式：当 `PoolId` 离开作用域时自动将槽位归还池中。这确保了即使发生 panic 或提前 return，ID 也不会泄漏。

## 设计要点

1. **代际安全的悬挂引用检测**：每次 `alloc` 从空闲池复用槽位时递增 `generation`。若外部代码持有旧的 `PoolId`，其 `generation` 与池中当前 `generation` 不匹配，可以通过比较检测到"已过时"的引用。这比纯索引方案更安全。

2. **RAII 自动回收**：通过 `PoolId` 的 `Drop` impl 自动归还 ID，消除了手动 `free` 忘记调用的风险。`IdPoolFree` 的 `Rc` 共享所有权允许多个 `PoolId` 引用同一个空闲列表。

3. **LIFO 空闲复用**：`free.0.borrow_mut().pop()` 使用栈式 LIFO 策略，倾向于复用最近释放的槽位，有更好的缓存局部性。

4. **条件复用**：`alloc_with_reuse_filter` 允许在分配层面实现"查找并复用匹配条件的已释放槽位"模式，减少了对象构造和分配开销。旧值通过返回值传递给调用者，允许额外的清理或状态检查。

5. **代际重置**：`alloc_new` 在新槽位上从 0 开始代际，而 `alloc` 复用时递增代际。这意味着代际值实际表示"此槽位被重新分配的次数"。
