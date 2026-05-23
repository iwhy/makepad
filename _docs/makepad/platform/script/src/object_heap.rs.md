# `object_heap.rs` — 对象堆操作

## 文件位置
`platform/script/src/object_heap.rs` (1221 行)

## 概述
`object_heap.rs` 在 `ScriptHeap` 上实现了所有与 **ScriptObject 对象**相关的操作，包括：
- 对象分配（new / new_with_proto / 复用空闲槽位）
- 对象标志位读写（freeze / deep / vec2 / auto / pod_type / type_index 等）
- 属性值设置（shallow / deep / checked 三种模式）
- 属性值读取（沿 prototype 链查找）
- 作用域（scope）读写
- vec 操作（push / pop / remove / insert / merge）
- map 操作（delete / len / iter）

---

## 方法详解

### 1. `new_object(&mut self) -> ScriptObject`
**分配**一个新对象。首先尝试从 `objects_free` 空闲列表中弹出已清空的槽位——这些槽位已在 GC 的 sweep 阶段被清理并递增了代号。如果复用成功，将 tag 设为 `alloced` 并将 `proto` 设为 `id!(object)`（默认原型）。如果空闲列表为空，则创建新槽位并 push 到 GenVec 中。

### 2. `new_with_proto_checked(&mut self, proto, trap) -> ScriptObject`
**带检查**的带原型对象创建。先检查 proto 参数是否为对象且未被标记为 `notproto`（不可用作原型），如果是则报错，否则委托给 `new_with_proto`。

### 3. `new_with_proto(&mut self, proto) -> ScriptObject`
**带原型**创建对象。委托给 `new_with_proto_impl(proto, true)`——要求从自动原型复制 vec 条目。

### 4. `new_with_proto_no_vec(&mut self, proto) -> ScriptObject`
**带原型但不复制 vec**。委托给 `new_with_proto_impl(proto, false)`。

### 5. `new_with_proto_impl(&mut self, proto, copy_vec_from_auto_proto) -> ScriptObject`
**核心实现**。逻辑如下：
1. 如果 proto 是对象，将其标记为 `reffed`（被引用），并获取其 `proto_fwd` 标记。
2. 如果 proto 不是对象，直接创建新对象并设置 proto。
3. 尝试从 `objects_free` 复用空闲槽位。复用后需要处理与 proto 对象的**别名安全**：通过 `slots_split_at_mut` 避免同时可变借用自身和 proto。
4. 设置 tag 为 `alloced` + `proto_fwd`。
5. 如果 `copy_vec_from_auto_proto` 为 true 且 proto 对象的 tag 标记为 `auto`，则将 proto 的 vec 内容**复制**到新对象中（即"自动展开"继承机制）。

### 6. `new_if_reffed(&mut self, ptr) -> ScriptObject`
**条件复制**。如果对象被标记为 `reffed`（有多处引用），则创建一个以相同 proto 为原型的新副本。否则直接返回原对象。这是实现**写时复制（Copy-on-Write）**的关键方法。

---

### 对象标志位操作

| 方法 | 功能 |
|---|---|
| `set_object_deep` | 标记为"深度"对象——属性设置时会遍历整个原型链 |
| `set_object_storage_vec2` | 标记为 vec2 存储模式——条目按 vec 而非 map 存储 |
| `set_object_storage_auto` | 标记为 auto 模式——子对象初始化时会自动复制当前 vec |
| `set_object_pod_type` | 在 tag 中记录关联的 POD 类型 |
| `set_first_applied_and_clean` | 标记已应用且为 clean 状态 |
| `clear_object_deep` | 清除 deep 标志 |
| `freeze` | 冻结对象，不可修改 |
| `set_notproto` | 标记为不可用作原型 |
| `set_from_eval` / `is_from_eval` | 标记/检查是否来自 eval 代码 |
| `freeze_module / component / shader / ext / api` | 不同冻结级别，对应模块/组件/着色器/扩展/API |
| `set_object_apply_transform` | 设置 apply 转换函数的 NativeId |
| `set_type` | 设置类型索引（用于类型检查） |
| `set_string_keys` | 标记为使用字符串键 |
| `set_shader_io` / `as_shader_io` | 设置/获取着色器 I/O 类型 |

---

### 属性值设置

#### `force_value_in_map(&mut self, ptr, key, value)`
**强制插入 map**。直接调用 `map_insert`，跳过所有检查。

#### `set_value_index(&mut self, ptr, index, value, trap) -> ScriptValue`
**按索引设置 vec 值**。将 `index` 作为 vec 的下标，如果索引超出 vec 当前长度则自动扩容。

#### `set_value_vec(&mut self, ptr, key, value, trap) -> ScriptValue`
**按键设置 vec 值**。从后往前遍历 vec，如果找到匹配的 key 则更新其 value；否则追加新条目。

#### `set_value_deep(&mut self, ptr, key, value, trap) -> ScriptValue`
**深度设置**（沿原型链）：
1. 从当前对象开始，遍历原型链。
2. 每到一个节点，先检查 vec 中是否有匹配 key，然后检查 map 中是否有匹配 key。
3. 找到后，如果节点不可变则报错；否则更新值。
4. 如果整条链都没找到，在最后一个节点上新增（根据 `is_vec2` 决定追加到 vec 还是 map）。

#### `set_value_shallow_checked(&mut self, top_ptr, key, key_id, value, trap) -> ScriptValue`
**带类型检查的浅层设置**：
1. 如果对象有 `type_index`，通过 `type_check` 中的 `props` 表查找字段的类型定义。
2. 如果找到类型定义，调用类型的 `check` 闭包验证值的合法性。如果不匹配，构造格式化的类型错误信息。
3. 如果字段不在类型中但对象允许 map_add，则新增到 map。
4. 如果字段不在类型中且不允许 map_add，则通过 `suggest_property` 提供拼写建议后报错。
5. 如果对象标记为 `validated`，沿原型链检查该属性是否存在且类型匹配（以原型上的值的类型为基准）。

#### `set_value_shallow(&mut self, ptr, key, value, trap) -> ScriptValue`
**浅层设置**（仅当前对象）：
- 如果 `is_vec2`：在 vec 中查找替换或追加。
- 否则：直接 map_insert。

#### `set_value_def(&mut self, ptr, key, value)`
**无陷阱默认设置**。调用 `set_value(ptr, key, value, NoTrap)`。

#### `set_value(&mut self, ptr, key, value, trap) -> ScriptValue`
**通用设置**。根据 key 类型分流：
- `LiveId` 键：
  - 如果对象不是 deep 模式但需要类型检查 → `set_value_shallow_checked`
  - 如果 `string_keys` 标记启用 → 将 LiveId 通过 `check_intern_string` 转为 intern 字符串值后浅设置
  - 常规 → `set_value_shallow`
  - deep 模式 → `set_value_deep`
- 数值索引键 → `set_value_index`
- 字符串/对象/颜色/bool 键 → 类似分流

#### `set_scope_value(&mut self, ptr, key, value, trap) -> ScriptValue`
**设置作用域变量**。从当前作用域对象开始沿原型链查找 key。如果找到则更新；如果整条链都找不到则报错（附带拼写建议 `suggest_scope_var`）。

---

### 属性值读取

#### `scope_value(&self, ptr, key, trap) -> ScriptValue`
**读取作用域变量**。沿原型链查找 `key`，先检查 map，再对 vec2 对象检查 vec，沿 proto 向上继续查找。

#### `def_scope_value(&mut self, ptr, key, value) -> Option<ScriptObject>`
**定义作用域变量**。如果当前作用域已存在该 key，则创建一个新作用域对象（以当前作用域为原型）并在新作用域中定义。否则直接在当前作用域的 map 中插入。返回 `Some(new_scope)` 表示发生了作用域分裂（shadowing）。

#### `value_index(&self, ptr, index, trap) -> ScriptValue`
**按索引读取 vec 值**。将索引作为 vec 下标读取。

#### `value_deep_map(&self, obj_ptr, key, trap) -> ScriptValue`
**仅 map 深度查找**。沿原型链只查找 map 中的 key（不查 vec）。

#### `value_deep(&self, obj_ptr, key, trap) -> ScriptValue`
**通用深度查找**。沿原型链：
1. 先查 map。
2. 处理 `string_keys` 与 `id` 之间的 JSON 互操作转换（双向适配）。
3. 查 vec（从后往前）。
4. 继续沿 proto 向上。

#### `object_method(&self, ptr, key, trap) -> ScriptValue`
**获取对象方法**。委托给 `value_deep_map`，因为方法在 map 中。

#### `value_path(&self, ptr, keys, trap) -> ScriptValue`
**按路径链取值**。遍历 `&[LiveId]` 路径，逐层通过 `obj -> value -> obj -> ...` 链式查找。如果中间某值不是对象则报错。

#### `value(&self, ptr, key, trap) -> ScriptValue`
**通用取值**。根据 key 类型分流：id → `value_deep`，数值索引 → `value_index`，字符串/对象/颜色/bool → `value_deep`。

---

### 原型字段（proto field）机制

#### `proto_field_from_type_check(&mut self, obj, field_id, trap) -> ScriptValue`
**从类型检查创建原型字段**。当通过 `obj.field` 访问一个只在类型检查结构中存在、不在具体对象上的字段时：
1. 通过 `field_type_from_type_check` 获取字段的类型 ID。
2. 通过 `type_default_for_id` 获取该类型的默认值对象。
3. 创建一个以默认值为原型的新对象。
4. 将新对象设置到当前对象上。
5. 实现了**深层次的原型继承**——访问时自动实例化。

#### `proto_field_from_value(&mut self, obj, field, trap) -> ScriptValue`
**从原型链值创建原型字段**。当字段值存在于原型链上但不在当前对象时：
1. 先检查当前对象是否已有该字段（直接 map 查找）。
2. 处理 `string_keys` 转换。
3. 如果没有，从原型链取值。
4. 如果值是对象，创建一个以该值为原型的新实例并设置到当前对象上。这实现了**自动实例化**——当读取原型链上的对象属性时，会创建一个专属副本。

---

### Apply 相关

#### `value_for_apply(&mut self, obj, key, apply) -> Option<ScriptValue>`
**为 apply 操作取值**。只检查当前对象和原型链上的 map（不查 vec），并且如果是 eval 操作，仅检查当前对象自身的 map，不回溯原型链。这是为了确保 apply 操作不会意外继承原型上的脏数据。

---

### Map 操作

| 方法 | 说明 |
|---|---|
| `map_ref(&self, object) -> &ScriptObjectMap` | 获取 map 的不可变引用 |
| `map_mut_with(&mut self, s, object, f) -> R` | 用 swap 模式获取 map 的可变访问——交换出 map，让闭包操作，再交换回来 |
| `map_delete(&mut self, ptr, key) -> Option<ScriptValue>` | 删除 map 条目 |
| `map_len(&self, ptr) -> usize` | map 条目数 |
| `iter_len(&self, ptr) -> usize` | 可迭代条目总数（vec 数 + map 数） |
| `iter_key_value(&self, ptr, index, trap) -> ScriptVecValue` | 统一迭代器——前 `vec.len()` 个索引来自 vec，之后来自 map |

---

### Vec 读取

| 方法 | 说明 |
|---|---|
| `vec_key_value(&self, ptr, index, trap)` | 读取 vec 中指定索引的 (key, value) 对 |
| `vec_value(&self, ptr, index, trap)` | 读取 vec 中指定索引的 value |
| `vec_value_if_exist(&self, ptr, index) -> Option<ScriptValue>` | 安全的可选读取 |
| `vec_len(&self, ptr) -> usize` | vec 条目数 |
| `vec_ref(&self, ptr) -> &[ScriptVecValue]` | vec 的切片引用 |

---

### Vec 写入

| 方法 | 说明 |
|---|---|
| `vec_insert_value_at` | 插入（当前为 stub，返回 NIL） |
| `vec_insert_value_begin` | 在开头插入（当前为 stub） |
| `vec_push_vec(&mut self, target, source, trap)` | 将一个对象的 vec 内容全部推入目标对象的 vec。通过 `slots_split_at_mut` 处理别名安全，检查目标是否 frozen |
| `vec_push_vec_of_vec(&mut self, target, source, map, trap)` | 遍历 source 的 vec 中的每个值，如果值是对象则将其 vec 内容推入 target。如果 map=true，还会合并 map。支持嵌套 push |
| `merge_object(&mut self, target, source, trap)` | 合并 source 到 target：推入 source 的 vec 条目，并仅添加 source 中 target 不存在的 map 条目（`merge_map_from_other_no_overwrite`） |
| `vec_push(&mut self, ptr, key, value, trap)` | 向 vec 追加一个条目 |
| `vec_push_unchecked(&mut self, ptr, key, value)` | 无检查追加 |
| `vec_remove(&mut self, ptr, index, trap)` | 删除 vec 中指定索引的条目 |
| `vec_pop(&mut self, ptr, trap)` | 弹出 vec 最后一个条目 |
