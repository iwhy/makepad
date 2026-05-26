# `handle.rs` — 句柄系统（外部 Rust 对象引用）

## 文件位置
`platform/script/src/handle.rs` (193 行)

## 概述
`handle.rs` 实现了 Makepad 脚本引擎的**句柄 (Handle) 系统**，这是脚本世界与 Rust 原生世界之间的桥梁。句柄允许脚本代码持有对 Rust 侧复杂对象（如纹理、网络连接、音频流等）的引用，同时不影响这些对象的 Rust 所有权和生命周期。

核心机制：
1. **`ScriptHandle`**：轻量级句柄标识符（类型 + 索引 + 代号的三元组）。
2. **`ScriptHandleData`**：句柄存储（tag + `Box<dyn ScriptHandleGc>`）。
3. **`ScriptHandleGc`**：trait，定义了句柄的 GC 清理和类型转换行为。
4. **`ScriptHandleRef`**：引用计数根包装，使句柄在 GC 标记阶段被保护。
5. **`HandleTag`**：句柄标记位（MARK 和 STATIC 标志）。

---

## 方法详解

### `ScriptHandleRef` 结构体

```
ScriptHandleRef {
    roots: Rc<RefCell<HashMap<ScriptHandle, usize>>>,
    handle: ScriptHandle,
}
```

引用计数根包装器。`ScriptHandleRef` 通过 `root_handles` 映射跟踪句柄的引用计数。当 `Clone` 时递增计数，当 `Drop` 时递减计数并可能在计数归零时移除条目。

#### `From<ScriptHandleRef> for ScriptValue`
**转换** `ScriptHandleRef` 为 `ScriptValue`。通过 `ScriptValue::from_handle(v.as_handle())` 实现。

#### `Clone for ScriptHandleRef`
**克隆**时递增引用计数。如果根映射中不存在该句柄（不应发生），输出错误日志。

#### `ScriptHandleRef::as_handle(&self) -> ScriptHandle`
**获取内部的 ScriptHandle**。

#### `Drop for ScriptHandleRef`
**析构**时递减引用计数。当计数降为 0 时从根映射中移除条目。这允许该句柄在下次 GC sweep 中被回收。

---

### `ScriptHeap` 上的句柄操作

#### `new_handle(&mut self, ty: ScriptHandleType, hgc: Box<dyn ScriptHandleGc>) -> ScriptHandle`
**创建新句柄**。执行流程：
1. 尝试从 `handles_free` 空闲列表复用槽位。复用时更新 `ty` 类型，调用 `hgc.set_handle(handle)` 通知句柄持有者自身的 `ScriptHandle`。
2. 如果空闲列表为空，push 新槽位到 `handles` GenVec（起始代号为 0）。
3. 创建 `ScriptHandleData { tag: Default::default(), handle: hgc }` 存储。
4. 返回句柄。

#### `handle_ref<T: ScriptHandleGc + 'static>(&self, handle: ScriptHandle) -> Option<&T>`
**获取句柄的不定类型引用**。通过 `downcast_ref::<T>()` 将 `dyn ScriptHandleGc` 转为具体的 `&T` 引用。如果类型不匹配或句柄已回收返回 `None`。

#### `handle_mut<T: ScriptHandleGc + 'static>(&mut self, handle: ScriptHandle) -> Option<&mut T>`
**获取句柄的可变不定类型引用**。通过 `downcast_mut::<T>()` 转换。用于修改句柄内部状态。

---

### `HandleTag` 标记位

```rust
pub struct HandleTag(u64);
```

使用位标志跟踪句柄状态：

| 标志 | 位 | 说明 |
|---|---|---|
| `MARK` | `0x1` | GC 标记位——sweep 时保留已标记的句柄 |
| `STATIC` | `0x2` | 静态句柄——永久存活，不被 GC 回收 |

方法：`is_marked()` / `set_mark()` / `clear_mark()` / `set_static()` / `is_static()`。

---

### `ScriptHandleData` 数据结构

```rust
pub struct ScriptHandleData {
    pub tag: HandleTag,           // 标记位
    pub handle: Box<dyn ScriptHandleGc>, // 动态句柄实现
}
```

#### `ScriptHandleData::gc(mut self)`
**调用句柄的 GC 清理**。在 GC sweep 阶段被回收的句柄会调用此方法，让句柄持有者有机会释放关联的外部资源（如关闭文件句柄、释放 GPU 内存等）。

---

### `ScriptHandleGc` Trait

句柄的**核心 trait**，由所有外部资源句柄实现：

| 方法 | 说明 |
|---|---|
| `gc(&mut self)` | 清理外部资源，在 GC sweep 回收此句柄时调用 |
| `set_handle(&mut self, handle)` | 设置句柄自身的 `ScriptHandle`，在创建后调用 |
| `ref_cast_type_id(&self) -> TypeId` | 返回具体的类型 ID，用于运行时类型识别 |
| `debug_fmt(&self, f) -> fmt::Result` | 调试输出格式化 |

#### `dyn ScriptHandleGc` 扩展方法

| 方法 | 说明 |
|---|---|
| `is::<T>() -> bool` | 检查句柄是否为类型 T |
| `downcast_ref::<T>() -> Option<&T>` | 向下转型为 &T（不安全的指针运算） |
| `downcast_mut::<T>() -> Option<&mut T>` | 向下转型为 &mut T（不安全的指针运算） |

`downcast_ref` 实现方式为：先通过 `is::<T>()` 做类型检查，然后通过 `unsafe { &*(self as *const dyn ScriptHandleGc as *const T) }` 直接指针转换。`downcast_mut` 类似。

`Debug for dyn ScriptHandleGc` 委托给 `debug_fmt` 方法。

---

## 在 GC 中的角色

句柄的生命周期与 GC 的关系：

1. **标记阶段**：在 `gc.rs` 的 `mark()` 函数中，`root_handles` 中的句柄被直接设置 `set_mark()`。此外，`mark_value_fields!` 宏会在遍历对象图时标记所有遇到的句柄。
2. **清除阶段**：在 `sweep()` 中，未标记的句柄被回收——调用 `handle_data.gc()` 释放外部资源，然后 `free_slot` 递增代号，并推入 `handles_free` 列表。
3. **静态句柄**：被标记为 static 的句柄跳过清除阶段，永久存活。
