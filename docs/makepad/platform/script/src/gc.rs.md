# `gc.rs` — 分代标记-清除垃圾收集器

## 文件位置
`platform/script/src/gc.rs` (773 行)

## 概述
`gc.rs` 实现了 Makepad 脚本引擎的**分代标记-清除 (Generational Mark-Sweep) 垃圾收集器**。该 GC 具有以下核心特性：

1. **分代标记-清除**：从根集合出发，遍历对象图标记所有可达对象，然后清理未标记（不可达）的对象。
2. **静态对象保护**：被标记为 `static` 的对象永久存活，不会被 GC 回收。
3. **根对象引用计数**：`ScriptObjectRef` / `ScriptArrayRef` / `ScriptHandleRef` 作为根引用，GC 从这些根出发进行标记。
4. **写屏障**（通过 `set_reffed` 机制）：当对象被引用时标记为 `reffed`，在确定无引用时通过 `free_object_if_unreffed` 提前回收。
5. **生长启发式触发**：类似 Lua 和 V8，当堆空间增长 2 倍时触发 GC。
6. **空闲槽位复用**：sweep 阶段回收的槽位递增代号后加入空闲列表，供后续分配复用。

---

## 核心数据结构

### `ScriptHeapGcLast`
记录上次 GC 后每种堆区的存活数量，用于 `needs_gc()` 的触发判断。

| 字段 | 类型 |
|---|---|
| `objects` | `usize` |
| `strings` | `usize` |
| `arrays` | `usize` |
| `pods` | `usize` |
| `handles` | `usize` |
| `regexes` | `usize` |

### `ScriptGcMark`
GC 工作列表中的条目，只有两种变体：

```rust
pub enum ScriptGcMark {
    Object(ScriptObject),
    Array(ScriptArray),
}
```

不需要为字符串/POD/句柄/正则表达式创建独立变体，因为它们被标记为静态时直接设置 `set_static()`，标记时直接设置 `set_mark()`——不会被加入工作列表。

---

## 宏

### `queue_static_val!($self, $val)`
在 `map_iter` 闭包内部使用，将值加入静态标记队列。由于闭包内无法进行 `is_static()` 检查，这个宏直接将所有引用值排队。

### `set_static_val!($self, $val)`
在非闭包环境中使用，先检查 `is_static()` 再决定是否排队，避免重复工作。

### `mark_value_fields!($objects, $arrays, $strings, $pods, $handles, $regexes, $mark_vec, $val)`
**核心标记宏**，使用**分散字段借用**（split field borrows）技术——分别传入 `$objects`、`$arrays` 等引用，避免借用整个 `ScriptHeap`。这使得调用者可以在遍历 map 的 for 循环中安全地调用此宏，而无需创建临时快照。

对不同类型的标记行为：
- **对象 (ScriptObject)**：如果非 static、已 alloced → 加入工作列表
- **字符串 (ScriptString)**：如果非 static → 直接 set_mark (不含或加入工作列表)
- **数组 (ScriptArray)**：同对象模式
- **POD (ScriptPod)**：如果非 static、已 alloced → 直接 set_mark
- **句柄 (ScriptHandle)**：跳过索引 0（空哨兵），非 static → set_mark
- **正则 (ScriptRegex)**：非 static → set_mark

---

## GC 两阶段

### 第一阶段：标记 (Mark)

#### `mark(&mut self, threads, code)`
**主标记函数**。执行步骤：

**A. 初始化**：清空 `mark_vec` 工作列表。

**B. 根集合枚举**，将所有可达的根加入工作列表：

1. **`type_check` 原型**：遍历 `self.type_check`，将每个已注册类型的 `object.proto` 加入标记——确保类型系统的原型对象不会被回收。
2. **`type_defaults` 默认值**：将所有类型的默认值对象加入工作列表。
3. **`pod_types` 默认值**：遍历所有 POD 类型，标记其 `default` 值和 `object` 字段。
4. **`root_objects` 根对象集**：遍历引用计数映射中所有 key，加入工作列表。这些是通过 `new_object_ref`/`ScriptObjectRef` 保持的根。
5. **`root_arrays` 根数组集**：同样处理。
6. **`root_handles` 根句柄集**：直接在 `handles` 上设 `set_mark()`。
7. **线程栈**：对每个脚本线程的：
   - `stack` 中的每个值
   - `scopes` 作用域对象
   - `mes` 方法调用的 self/args/pod/array
   - `loops` 循环的 source 值
   - `trap.err` 中的错误对象值
   - `trap.on` 中的 Return/Bail 值
8. **代码体 (ScriptBody)**：
   - `ScriptSource::Mod` 中的 `values` 数组
   - tokenizer 中的字符串字面量（作为根保护）
9. **原生类型表**：`code.native.type_table` 中的对象

**C. 工作列表处理**：用 `while` 循环处理 `mark_vec` 中的每个条目，因为 `mark_inner` 处理过程中可能向队列追加新条目。这正是**遍历回溯**的工作列表模式。

#### `mark_inner(&mut self, mark: ScriptGcMark)`
**标记单个条目并展开其引用**。使用**分散字段借用**技术在无需快照的情况下遍历对象的 map 和 vec：

- **对象**：如果已 static / 已 marked / 未 alloced → 跳过。否则设 `set_mark()`，然后遍历：
  - `proto` 原型值
  - `map` 中的每个 key 和 value
  - `vec` 中的每个 key 和 value
  对每个引用的对象或数组，如果非 static 且已 alloced，加入工作列表。
- **数组**：类似处理。如果是 `ScriptArrayStorage::ScriptValue`，遍历所有元素值。

#### `mark_value(&mut self, val: ScriptValue)`
**标记单个值**。内联版本的辅助函数，使用分散字段借用直接调用 `mark_value_fields!` 宏。

---

### 第二阶段：清除 (Sweep)

#### `sweep(&mut self, log_stats: bool)`
**清除阶段**。遍历每种堆区的所有槽位（从索引 1 开始，索引 0 保留），决定每个槽位的命运：

**清理规则**：
1. **static** 对象 → 跳过（永久存活），清除其 mark 标记。
2. **alloced 但未 marked** → 不可达，回收。
3. **alloced 且 marked** → 存活，清除 mark 标记。

**各堆区回收逻辑**：

| 堆区 | 回收行为 |
|---|---|
| **objects** | 如果关联了 pod_type，将其推入 `pod_types_free` 回收。调用 `object.clear()` 清空数据。`free_slot` 递增代号。将新代号的对象推入 `objects_free`。如果有对象被移除，调用 `bump_object_reuse_epoch()` 通知上层。 |
| **arrays** | 调用 `array.clear()`。`free_slot` 递增代号。推入 `arrays_free`。 |
| **strings** | 从 `string_intern` 驻留表中移除。将 `Arc::into_inner` 获得的 `String` 归还到 `strings_reuse` 池。`free_slot` 递增代号。推入 `strings_free`。 |
| **handles** | 调用 `handle_data.gc()` 通知句柄进行自己的清理。`free_slot` 递增代号。推入 `handles_free`。 |
| **pods** | 调用 `pod.clear()`。`free_slot` 递增代号。推入 `pods_free`。 |
| **regexes** | 从 `regex_intern` 驻留表中移除（重建 intern key）。`free_slot` 递增代号。推入 `regexes_free`。 |

**统计与日志**：
- 对每种堆区统计：static 数、存活数、移除数
- 如果 `log_stats=true`，输出单行 GC 统计信息：`GC {}us: obj[S:{} A:{} R:{}] arr[...] str[...] hdl[...] pod[...] rex[...]`

**更新 `gc_last`**：记录清除后的存活数量，供 `needs_gc()` 使用。

---

### 静态标记系统

#### `set_static(&mut self, value: ScriptValue)`
**将值及其所有可达对象标记为静态**（永久存活）。从根值开始，将工作列表初始推入 `mark_vec`，然后用 `while` 循环逐项处理。

#### `set_static_inner(&mut self, value: ScriptGcMark)`
**单个静态标记**。如果对象/数组未 static 且已 alloced：
1. 设 `set_static()`。
2. 递归标记 proto、map 条目、vec 条目的所有引用值。

---

### 根引用系统

#### `new_object_ref(&mut self, obj: ScriptObject) -> ScriptObjectRef`
**创建对象根引用**。标记对象为 `reffed`，在 `root_objects` 映射中递增引用计数。返回 `ScriptObjectRef`，当此值 drop 时会递减计数。

#### `new_array_ref(&mut self, array: ScriptArray) -> ScriptArrayRef`
**创建数组根引用**。类似 `new_object_ref`，但操作 `root_arrays`。

#### `new_fn_ref(&mut self, obj: ScriptObject) -> ScriptFnRef`
**创建函数根引用**。包装了 `new_object_ref`。

#### `new_handle_ref(&mut self, handle: ScriptHandle) -> ScriptHandleRef`
**创建句柄根引用**。操作 `root_handles` 映射，递增引用计数。

---

### GC 触发策略

#### `needs_gc(&self) -> bool`
**判断是否需要触发 GC**。使用基于生长因子的启发式（类似 Lua 和 V8）：
- 每种堆区有最小触发阈值（防止小堆时频繁 GC）：
  - Objects: 1024, Strings: 256, Arrays: 128, Pods: 128, Handles: 64
- 生长因子为 **2×**：当前存活数 ≥ 上次 GC 后存活数的 2 倍
- 任一堆区超过阈值且超过生长因子 → 返回 true

#### `gc_live_len(&self) -> usize`
**计算当前总存活数**。从每种堆区的总长度中减去空闲列表长度，求和。用于外部监控。

---

### 写屏障/提前回收

#### `free_object_if_unreffed(&mut self, ptr: ScriptObject)`
**如果对象没有被引用则提前回收**。这是写屏障的核心：
1. 通过 `is_valid` 检查引用是否仍有效（可能已被 GC 回收）。
2. 如果对象已 alloced 但未 `reffed`（没有任何根引用或原型链引用）：
   - 回收其关联的 pod_type（如果有）。
   - 调用 `clear()` 清空数据。
   - `free_slot` 递增代号。
   - 推入 `objects_free` 列表。
   - `bump_object_reuse_epoch()`。

此机制确保当某个对象的唯一引用被释放时（如 `ScriptObjectRef` drop 后引用计数降为 0），该对象可以**立即回收**而不必等待下一次完整 GC。
