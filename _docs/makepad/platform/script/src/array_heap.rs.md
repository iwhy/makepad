# `array_heap.rs` — 数组堆操作

## 文件位置
`platform/script/src/array_heap.rs` (274 行)

## 概述
`array_heap.rs` 在 `ScriptHeap` 上实现了所有与 **ScriptArray 数组**相关的操作。数组支持多种存储类型 (`ScriptArrayStorage`)：
- `ScriptValue` — 通用脚本值数组（最常用）
- `U8 / U16 / U32` — 无符号整数紧凑存储
- `F32` — 浮点数紧凑存储

每个操作都包含**不可变检查**和**脏标记**机制。

---

## 方法详解

### 1. `freeze_array(&mut self, array: ScriptArray)`
**冻结数组**。调用 `self.arrays[array].tag.freeze()`，使数组变为只读。后续所有修改操作都会失败。

### 2. `new_array(&mut self) -> ScriptArray`
**分配新数组**。执行流程：
1. 尝试从 `arrays_free` 空闲列表复用槽位（槽位已在 GC sweep 中清理并递增了代号）。
2. 如果复用的是**类型化数组槽位**（U8/U16/U32/F32），需要重置为 `ScriptValue` 存储——因为复用槽位必须从通用存储开始。
3. 如果复用槽位已经是 `ScriptValue` 存储，只需清空。
4. 如果空闲列表为空，创建新槽位 push 到 `arrays` GenVec（起始代号为 0）。
5. 设置 tag 为 `alloced`。
6. 测试用例：`reused_array_slot_resets_to_script_value_storage` 验证了复用逻辑。

### 3. `array_len(&self, array: ScriptArray) -> usize`
**获取数组长度**。调用 `self.arrays[array].storage.len()`，与存储类型无关。

### 4. `array_push(&mut self, array, value, trap)`
**追加元素**。检查不可变 → 设置脏标记 → `storage.push(value)`。

### 5. `array_pop_front_option(&mut self, array) -> Option<ScriptValue>`
**弹出首元素**。检查不可变 → 设置脏标记 → `storage.pop_front()`（对应 `VecDeque::pop_front` 语义）。

### 6. `array_push_vec(&mut self, array, object, trap)`
**将对象的 vec 条目追加到数组**。遍历 `object.vec` 中的每个 value，逐一 `storage.push`。

### 7. `merge_array(&mut self, target, source, trap)`
**合并源数组到目标数组**。将 source 中的所有元素（无论存储类型）按顺序收集为 `ScriptValue` 列表，然后逐一推入 target。这是 **splat 运算符 `..`** 的数组一侧实现。支持所有存储类型间的自动转换。

### 8. `array_push_unchecked(&mut self, array, value)`
**无检查追加**。直接设置脏标记并 push。跳过不可变检查（用于内部已知安全的操作）。

### 9. `array_storage(&self, array: ScriptArray) -> &ScriptArrayStorage`
**获取存储的不可变引用**。

### 10. `new_array_from_vec_u8(&mut self, data: Vec<u8>) -> ScriptArray`
**从 u8 向量创建类型化数组**。创建新数组后直接替换 storage 为 `ScriptArrayStorage::U8(data)`，并设置脏标记。用于二进制数据（如文件内容、纹理数据）。

### 11. `array_mut(&mut self, array, trap) -> Option<&mut ScriptArrayStorage>`
**获取存储的可变引用**。检查不可变 → 设置脏标记 → 返回 `&mut storage`。

### 12. `array_mut_self_with<R, F>(&mut self, array, cb) -> R`
**通过 swap 模式执行数组不可变读取**。将 storage swap 出，让闭包在只读模式下（`&ScriptArrayStorage`）操作，然后 swap 回。这允许闭包在持有 `&self` 的情况下安全地读取数组内容而不违反借用规则。

### 13. `array_mut_mut_self_with<R, F>(&mut self, array, cb) -> R`
**通过 swap 模式执行数组可变操作**。与上述类似，但闭包接收 `&mut ScriptArrayStorage`。用于需要对数组存储进行结构修改的安全包装。

### 14. `array_remove(&mut self, array, index, trap) -> ScriptValue`
**删除指定索引的元素**。边界检查 → `storage.remove(index)`。所有后续元素前移。

### 15. `array_pop(&mut self, array, trap) -> ScriptValue`
**弹出末尾元素**。从 `storage.pop()` 获取最后一个元素。空数组报错。

### 16. `array_clear(&mut self, array, trap)`
**清空数组**。仅当数组非空时执行 `storage.clear()` 并设置脏标记。

### 17. `array_index(&self, array, index, trap) -> ScriptValue`
**带边界检查的索引读取**。调用 `storage.index(index)`，该函数统一了所有存储类型的索引接口。越界时报错。

### 18. `array_index_unchecked(&self, array, index) -> ScriptValue`
**无检查索引读取**。越界时返回 `NIL` 而非报错。用于内部已知安全的访问。

### 19. `set_array_index(&mut self, array, index, value, trap) -> ScriptValue`
**设置指定索引的值**。检查不可变 → 设置脏标记 → `storage.set_index(index, value)`。用于支持数组元素的双向绑定和即时更新。
