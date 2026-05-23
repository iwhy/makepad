# draw_matrix.rs — 变换矩阵堆栈

## 概述

`draw_matrix.rs` 实现了 Makepad 绘制管线中的**变换矩阵堆栈管理**。它使用世代编号池 (`IdPool`) 管理 `Mat4f` 矩阵节点，支持局部/父级矩阵的增量更新和版本追踪，为场景图的坐标变换系统提供基础支持。

---

## `DrawMatrix` / `DrawMatrixId`（第 4–8 行）

`DrawMatrix` 是一个轻量级句柄，包裹 `PoolId`。`DrawMatrixId` 包含 `(usize, u64)` 二元组（池索引 + 世代编号），防止悬垂引用。

**`DrawMatrix::identity()`** — 返回 `DrawMatrixId(0, 0)` 作为单位矩阵节点的特殊 ID。

---

## `CxDrawMatrix`（第 10–20 行）

矩阵节点的内部状态：
- `parent: DrawMatrixId` — 父矩阵节点的 ID
- `parent_version: u64` — 父节点的版本号，用于检测父节点变化
- `local_version: u64` — 局部矩阵的版本号，每次 `local` 变化时递增
- `local: Mat4f` — 局部变换矩阵（相对父节点的变换）
- `object_to_world: Mat4f` — 缓存的对象到世界矩阵
- `world_to_object: Option<Mat4f>` — 可选的世界到对象逆矩阵缓存

---

## `CxDrawMatrixPool`（第 40–83 行）

矩阵池管理器，使用 `IdPool<CxDrawMatrix>` 分配矩阵节点：

- **`alloc()`** — 分配一个新的矩阵节点，返回 `DrawMatrix` 句柄。
- **`update_local()`** — 更新指定节点的局部矩阵：
  - 比较新矩阵与当前 `local` 是否相等。
  - 若不同则递增 `local_version` 并写入新值。
  - 版本号用于下游节点判断是否需要级联更新。

- **`update_parent()`** — 更新父节点关系（当前为占位实现，仅在非 identity 节点时检查父节点版本）。

---

## 默认构造（第 45–52 行）

`CxDrawMatrixPool::default()` 的行为：
1. 创建一个 `IdPool`。
2. 立即分配索引 0 作为恒等矩阵（identity）。
3. 所有后续分配从索引 1 开始。

这使得 `DrawMatrixId(0, 0)` 始终代表单位矩阵，可作为全局共享根使用。

---

## 索引访问（第 85–110 行）

`Index<DrawMatrixId>` 和 `IndexMut<DrawMatrixId>` 的实现从池中安全地读取矩阵节点，包含世代编号校验：

```rust
if d.generation != index.1 {
    error!("MatrixNode id generation wrong {} {} {}", index.0, d.generation, index.1)
}
```

这是一种防止 use-after-free 的安全措施——若池中该槽位已被回收并重用，世代编号会不匹配，触发错误。

---

## 设计意图

当前实现主要提供了矩阵节点的**分配、ID 管理和局部矩阵更新**功能。注释中留下了 `get_object_to_world()` 的占位（第 79–82 行），暗示未来的按需级联更新逻辑将从父节点递归计算 `object_to_world` 矩阵。

`update_parent()` 函数体中的注释（第 71–77 行）显示了一个基于版本号的惰性更新策略：子节点在访问时比较自己的 `parent_version` 与父节点的 `local_version`，若不一致则重新计算自己的 `object_to_world`。
