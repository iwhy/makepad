# `gen_index.rs` 源码解读

**路径:** `platform/script/src/gen_index.rs`
**行数:** 283
**核心职责:** 实现代际索引（Generational Index）系统，通过在每次槽位释放时递增代数计数器来检测悬挂引用（use-after-free）bug，在 release 构建下可通过 feature flag 关闭检查。

---

## Trait `GenRef`

**路径:** 第12-16行

定义代际引用的基本接口：
- `index()` 返回槽位索引（u32）
- `generation()` 返回当前代际值
- `new()` 根据索引和代际构造引用类型

此 trait 的每个实现者都是一个轻量级句柄，携带 (index, generation) 对。

---

## 结构体 `GenSlot<T>`

**路径:** 第19-64行

每个槽位的元数据容器：
- `generation`（仅在 `check_gen` feature 下存在）：代际计数器。
- `data`：实际存储的数据。

### 方法
- **`new(data)`**：创建新槽位，generation 初始化为 0。
- **`increment_generation()`**：代际递增（wrapping add 1）。无检查模式下为空操作。

---

## 结构体 `GenVec<T>`

**路径:** 第72-220行

一个带有代际跟踪的向量容器。既支持通过 `GenRef` 实现者的检查访问，也支持通过原始 `usize` 索引的非检查内部访问。

### 构造方法
- **`new()` / `with_capacity(capacity)`**：创建空向量或预分配容量。

### 生命周期管理
- **`push(data)`**：追加新元素，返回 `(index, generation)`。无检查模式下 generation 为 `()`。
- **`free_slot(index)`**：释放槽位，递增代际计数器。
- **`allocate<R>(free_list)`**：首选从空闲列表回收槽位（保留已递增的代际），否则扩容。
- **`is_valid<R>(r)`**：验证引用是否有效（索引越界或代际不匹配则无效）。无检查模式下只检查索引范围。

### 查询
- **`len()` / `is_empty()`**：长度查询。
- **`generation(index)`**：获取指定槽位的代际值。

### 迭代
- **`iter()` / `iter_mut()`**：不可变/可变迭代器，仅返回数据部分，隐藏代际字段。
- **`slots_split_at_mut(mid)`**：在 mid 处切分为两个可变切片，用于同时对相邻槽位做不同操作。

### 内部方法（绕过代际检查）
- **`get_at(index)` / `get_at_mut(index)`**：通过原始索引直接访问（GC 遍历用）。
- **`set_at(index, value)`**：直接设置（GC sweep 用）。

---

## 函数 `check_generation`

**路径:** 第225-237行

仅在 `check_gen` feature 下编译。当槽位代际与引用代际不匹配时 panic，提供详细的错误信息（类型名、索引、代际值）。

---

## Index / IndexMut 实现

### `Index<R>`（第240-261行）
- 通过 `GenRef` 做检查访问：先查代际，再返回数据引用。

### `IndexMut<R>`（第264-283行）
- 通过 `GenRef` 做检查可变访问：先查代际，再返回数据可变引用。
