# `heap.rs` — ScriptHeap 分代堆管理器

## 文件位置
`platform/script/src/heap.rs` (801 行)

## 概述
`ScriptHeap` 是 Makepad 脚本引擎的中央堆管理器，统一管理六大堆区：**对象 (Object)**、**字符串 (String)**、**数组 (Array)**、**POD (Plain Old Data)**、**句柄 (Handle)** 和 **正则表达式 (Regex)**。每个堆区使用 `GenVec`（带代号的生成式向量）来实现**世代安全引用**——每次槽位被回收时递增其代号，从而让悬空指针在访问时被检测到。

除了堆区管理，`ScriptHeap` 还负责：
- 类型注册与类型检查（`type_check` / `type_index`）
- 根对象集（`root_objects` / `root_arrays` / `root_handles`），供 GC 标记使用
- 值到各种类型的转换（`cast_to_f64` / `cast_to_bool`）
- 深度相等比较（`deep_eq`）
- 调试输出（`to_debug_string` / `println`）
- JSON 序列化（`to_json` / `to_json_inner`）
- 对象复用纪元计数器（`object_reuse_epoch`），供上层缓存失效使用

---

## 核心数据结构

### `ScriptHeap` 字段

| 字段 | 类型 | 用途 |
|---|---|---|
| `modules` | `ScriptObject` | 顶层模块对象，所有模块的根 |
| `gc_last` | `ScriptHeapGcLast` | 上次 GC 后各堆区的存活计数，用于判断触发下次 GC |
| `mark_vec` | `Vec<ScriptGcMark>` | GC 标记阶段的工作列表 |
| `object_reuse_epoch` | `u64` | 对象槽位被释放/复用时的单调递增计数器 |
| `root_objects/arrays/handles` | `Rc<RefCell<HashMap<...>>>` | 根对象引用计数集合，GC 从此出发标记 |
| `type_defaults` | `HashMap<ScriptTypeIndex, ScriptObject>` | 类型默认值对象表 |
| `objects` | `GenVec<ScriptObjectData>` | 对象主存储（生成式向量） |
| `objects_free` | `Vec<ScriptObject>` | 空闲对象槽位列表 |
| `string_intern` | `HashMap<ScriptRcString, ScriptString>` | 字符串常量表（驻留 intern） |
| `strings_reuse` | `Vec<String>` | 可复用的临时字符串缓冲区池 |
| `strings` | `GenVec<Option<ScriptStringData>>` | 字符串主存储 |
| `strings_free` | `Vec<ScriptString>` | 空闲字符串槽位列表 |
| `arrays` 等 | 同模式 | 数组堆区 |
| `pod_types` | `Vec<ScriptPodTypeData>` | POD 类型描述符存储 |
| `pod_types_free` | `Vec<ScriptPodType>` | 空闲 POD 类型槽位 |
| `pods` | `GenVec<ScriptPodData>` | POD 数据主存储 |
| `pods_free` | `Vec<ScriptPod>` | 空闲 POD 槽位 |
| `type_check` | `Vec<ScriptTypeCheck>` | 已注册的类型检查器列表 |
| `type_index` | `HashMap<ScriptTypeId, ScriptTypeIndex>` | Rust TypeId 到 ScriptTypeIndex 的映射 |
| `handles` | `GenVec<Option<ScriptHandleData>>` | 句柄主存储 |
| `handles_free` | `Vec<ScriptHandle>` | 空闲句柄槽位 |
| `regex_intern` | `HashMap<RegexInternKey, ScriptRegex>` | 正则表达式驻留表 |
| `regexes` | `GenVec<Option<ScriptRegexData>>` | 正则表达式主存储 |
| `regexes_free` | `Vec<ScriptRegex>` | 空闲正则槽位 |

### 槽位 0 约定
所有堆区的 **槽位 0** 保留为空/空值哨兵。`ScriptObject::ZERO`、`ScriptString::ZERO` 等指向此处，表示"空引用"。在 `empty()` 初始化时，每个 GenVec 都会先 push 一个槽位 0 并冻结它。

---

## 方法详解

### 1. `empty() -> Self`
**初始化**一个空的 ScriptHeap。执行以下步骤：
1. 创建 6 个 `GenVec`（objects/arrays/pods/handles/strings/regexes），每个都先 push 一个槽位 0。
2. 将对象槽位 0 标记为 `alloced + static + frozen`，数组槽位 0 标记为 `alloced + frozen`——这样它们永远不会被 GC 回收。
3. 创建一个顶级模块对象（以 `id!(mod)` 为原型），将其注册为根对象，赋值给 `self.modules`。

### 2. `registered_type(&self, id: ScriptTypeId) -> Option<&ScriptTypeCheck>`
**查询**已注册的类型检查器。通过 `type_index` 哈希表将 `ScriptTypeId`（Rust 的 `TypeId`）映射到 `type_check` 数组的索引，返回对应的类型检查结构体。

### 3. `register_type(&mut self, type_id: Option<ScriptTypeId>, ty_check: ScriptTypeCheck) -> ScriptTypeIndex`
**注册**一个新的类型。分配一个新的 `ScriptTypeIndex`（等于 `type_check.len()` 处的序号），如果传入了 `type_id` 则将其插入 `type_index` 映射表，然后将类型检查结构体推入 `type_check` 向量。返回分配的索引。

### 4. `type_matches_id(&self, ptr: ScriptObject, type_id: ScriptTypeId) -> bool`
**校验**对象是否匹配给定的类型 ID。先获取对象的 tag 中的 `type_index`，然后通过 `type_check` 数组查询对应的 `object.type_id` 是否与参数相等。

### 5. `object_type_id(&self, ptr: ScriptObject) -> Option<ScriptTypeId>`
**获取**对象的类型 ID。从对象的 tag 中提取 `type_index`，通过 `type_check` 数组取出类型检查结构体中的 `object.type_id`。

### 6. `type_name_by_id(&self, type_id: ScriptTypeId) -> Option<LiveId>`
**获取**类型注册名。通过 `type_index` 找到类型检查结构体，返回其 `object.name` 字段。

### 7. `new_module(&mut self, id: LiveId) -> ScriptObject`
**创建**一个新模块。创建一个以 `id` 为原型的新对象，然后将其作为属性设置到 `self.modules` 下，成为全局模块树的一部分。

### 8. `module(&mut self, id: LiveId) -> ScriptObject`
**获取**已注册的模块对象。从 `self.modules` 中按 `id` 键查找值并转换为 `ScriptObject`。

### 9. `has_proto(&mut self, ptr: ScriptObject, rhs: ScriptValue) -> bool`
**检查**原型链中是否存在指定值。从 `ptr` 开始，沿 `proto` 链向上遍历，判断 `rhs` 是否是链上的某个原型。此方法用于运行时判断对象是否属于某个类型。

### 10. `proto(&self, ptr: ScriptObject) -> ScriptValue`
**获取**对象的直接原型。

### 11. `root_proto(&self, ptr: ScriptObject) -> ScriptValue`
**获取**原型链最顶端的根原型。沿 `proto` 链一直向上追溯到尽头，返回最后一个非对象原型值。

### 12. `object_data(&self, ptr: ScriptObject) -> &ScriptObjectData`
**获取**对象数据的不可变引用。

### 13. `object_reuse_epoch() -> u64` / `bump_object_reuse_epoch()`
**对象复用纪元**计数器。当对象槽位被回收复用时，`bump` 会使计数器递增。上层缓存（如类型缓存）可通过检查此纪元是否变化来决定是否清空缓存。

### 14. `type_check(&self, index: ScriptTypeIndex) -> &ScriptTypeCheck`
**获取**给定索引的类型检查器引用。

### 15. `set_type_default(&mut self, obj: ScriptObject) -> bool`
**设置**类型默认值对象。如果对象 tag 中包含 `type_index`，则将其记录到 `type_defaults` 表中。GC 阶段会扫描此表以保护默认值不被回收。

### 16. `type_default(&self, ty_index: ScriptTypeIndex) -> Option<ScriptObject>`
**获取**指定类型索引的默认值对象。

### 17. `type_default_for_id(&self, type_id: ScriptTypeId) -> Option<ScriptObject>`
**获取**指定类型 ID 的默认值对象。先通过 `type_index` 找到 `ScriptTypeIndex`，再查 `type_defaults` 表。

### 18. `field_type_from_type_check(&self, obj: ScriptObject, field_id: LiveId) -> Option<ScriptTypeId>`
**获取**类型检查结构中某个字段的类型 ID。先取对象的 `type_index`，在 `type_check` 数组中通过 `props.props` 哈希表查找字段 `field_id`，返回其类型 ID。如果当前对象没找到，会沿原型链递归查找。用于实现深层次的原型继承类型推导。

### 19. `cast_to_f64(&self, v: ScriptValue, ip: ScriptIp) -> f64`
**值 → f64 转换**。依次尝试将值转为：
- 如果本身是 f64，直接返回
- u40 → 整数提升
- 字符串 → `parse::<f64>()`
- bool → 1.0 / 0.0
- f32 / f16 → 精度提升
- u32 / i32 → 整数提升
- color → 整数作为 f64
- nil → 0.0
- 否则 → `from_f64_traced_nan(f64::NAN, ip)` 追踪 NaN 的来源（用于调试）

### 20. `cast_to_bool(&self, v: ScriptValue) -> bool`
**值 → bool 转换**。依次尝试：
- 本身就是 bool → 直接返回
- nil → false
- f64/f32/f16/u40/u32/i32 → 非零则为 true
- object → true
- 内联字符串非空 → true
- 堆字符串长度 > 0 → true
- id/color/opcode → true

### 21. `deep_eq(&self, a: ScriptValue, b: ScriptValue) -> bool`
**深度相等比较**。处理以下情况：
- 指针相等 (`a == b`) → true
- 数值比较（绕过 NaN 追踪包装）
- **对象比较**：沿 prototype 链递归比较每个 `vec` 条目和 `map` 条目。处理 `string_keys` 与 `id` 之间的 JSON 互操作转换（字符串键 vs LiveId 键的双向适配）
- **数组比较**：区分存储类型 (`ScriptValue` / F32 / U32 / U16 / U8)，逐元素比较

### 22. `println(&self, value: ScriptValue)`
**打印**值到控制台。将值格式化为字符串后调用 `println!`。

### 23. `to_debug_string(&self, value, recur, out, formatted, depth)`
**格式化**值为调试字符串。这是核心格式化函数，处理：
- **对象**：输出 `<index>{...}` 格式，显示原型链（`^<proto_index>`）。如果是函数类型，输出 `<fn index>`。循环引用检测通过 `recur` 向量跟踪已展开的值。
- **类型信息**：如果对象有 `type_index`，输出 `<type field1, field2>` 标记。
- **数组**：输出 `<index>[elem1, elem2, ...]`。
- **字符串**：输出 `"内容"`（包括内外引号）。
- **POD**：调用 `pod_debug` 输出。
- **其他值**：使用 `Display` trait 输出。
`formatted` 参数控制是否使用多行缩进格式（用于 `println`）还是紧凑格式。

### 24. `to_json(&mut self, value: ScriptValue) -> ScriptValue`
**将值序列化为 JSON 字符串**。创建一个新字符串，调用 `to_json_inner` 写入。

### 25. `to_json_inner(&self, value: ScriptValue, out: &mut String)`
**JSON 序列化核心**：
- **对象** → `{key:value,...}`，遍历 map 和 vec，沿 prototype 链往上合并属性
- **数组** → `[elem,...]`，支持所有存储类型（ScriptValue/F32/U32/U16/U8）
- **ID** → `"id_string"`，通过 `LiveId::as_string` 尝试还原字符串
- **字符串** → `"转义内容"`，处理 `\b`、`\f`、`\n`、`\r`、`"`、`\\` 转义
- **bool** → `true` / `false`
- **数值** → 直接输出
- **句柄** → `Handle{:?}`
- 其他 → `null`

### 26. `objects_len(&self) -> usize`
**返回**对象堆区的总长度（含空闲槽位）。

### 27. `has_apply_transform(&self, value: ScriptValue) -> bool`
**检查**值是否具有 apply 变换。如果是对象或数组，检查其 tag 中是否设置了 `apply_transform`。此方法用于类型检查系统——当值有 apply 变换时，类型检查可以更宽松。
