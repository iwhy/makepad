# geometry.rs — 几何体数据管理

## 概述

`geometry.rs` 管理 Makepad 绘制管线中的**几何体数据**（顶点和索引缓冲区）。它提供 `Geometry` 句柄、`GeometryId` 世代编号 ID、`CxGeometryPool` 池化存储以及高效的缓冲区更新方法，支持零拷贝交换以减少内存分配。

---

## `Geometry` / `GeometryId`（第 4–13 行）

`Geometry` 是一个轻量级句柄，包裹 `PoolId`（分配自 `IdPool<CxGeometry>`）。`GeometryId` 包含 `(usize, u64)` 二元组，`usize` 为池索引，`u64` 为世代编号，用于检测悬垂引用。

**`Geometry::geometry_id()`** — 从内部 PoolId 提取 `GeometryId`。

---

## `ScriptHandleGc` 实现（第 6–10 行）

`Geometry` 实现了 `ScriptHandleGc` trait。当脚本 GC 回收该句柄时，`gc()` 方法调用 `self.0.free()` 释放池中的槽位。

---

## `CxGeometryPool`（第 33–67 行）

`CxGeometryPool` 是对 `IdPool<CxGeometry>` 的包装，提供：
- **`alloc()`** — 分配新的几何体槽位。

**`Index<GeometryId>` / `IndexMut<GeometryId>`** — 通过 `GeometryId` 访问 `CxGeometry`，包含世代编号验证：
```rust
if d.generation != index.1 {
    error!("Drawlist id generation wrong {} {} {}", index.0, d.generation, index.1)
}
```
若世代不匹配则记录错误——这是针对 use-after-free 的保护。

---

## `Geometry` 公有方法（第 69–121 行）

### `into_script_handle()`（第 70–74 行）

将 `Geometry` 转换为脚本堆中的句柄值：
1. 通过 `vm.handle_type(id!(geometry))` 获取几何体句柄类型。
2. 使用 `heap.new_handle()` 将 `self` 装箱到堆中。
3. 返回 `ScriptValue::Handle`。

### `Geometry::new(cx)`（第 76–84 行）

创建新几何体，分配池槽位并初始化所有脏标记为 true：
- `indices` 和 `vertices` 被清空
- `dirty`、`dirty_vertices`、`dirty_indices` 均设为 true

### `Geometry::update(cx, indices, vertices)`（第 86–93 行）

**标准更新方法**：将传入的索引和顶点数据赋给几何体，设置所有脏标记。此处发生数据的克隆/所有权转移。

### `Geometry::update_with_recycled_buffers(cx, indices, vertices)`（第 99–113 行）

**零拷贝更新方法**（推荐用于每帧更新的几何体）：
1. 通过 `std::mem::swap` 将传入的 vec 与几何体的 vec 交换。
2. 清空传入的 vec（复用容量，下次使用无需重新分配）。
3. 设置脏标记为 true。

这种方法的关键收益：调用方可复用已清空的 vec 来构造下一帧的数据，避免连续帧之间的重复内存分配和释放。

### `Geometry::update_indices(cx, indices)`（第 115–121 行）

仅更新索引数据，保留已有的顶点数据：
- 设置 `dirty_indices = true` 和 `dirty = true`
- 不设置 `dirty_vertices`（保持 false，避免不必要上传）

---

## `CxGeometry`（第 124–132 行）

几何体的运行时状态：
- `indices: Vec<u32>` — 索引缓冲区（u32 整数）
- `vertices: Vec<f32>` — 顶点缓冲区（f32 浮点数，通常为交错布局）
- `dirty: bool` — 通用脏标记
- `dirty_vertices: bool` — 顶点数据需要上传到 GPU
- `dirty_indices: bool` — 索引数据需要上传到 GPU
- `os: CxOsGeometry` — 平台相关的几何体后端数据（如 OpenGL VAO/VBO ID）

三个脏标记的分离设计允许在上传时精确控制 GPU 缓冲区更新：仅上传变化的缓冲区以减少带宽消耗。
