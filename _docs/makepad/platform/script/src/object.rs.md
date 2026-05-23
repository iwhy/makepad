# `object.rs` — 动态对象系统：`ScriptObject` 与双通道属性存储

## 文件位置
`platform/script/src/object.rs` (1087 行)

## 核心类型

### `ScriptObjectTag(u64)` — 64 位标志位标签
使用一个 `u64` 的 **高 24 位**编码该对象的各类元信息，低 40 位供 `REF_DATA_MASK` 使用。这是整个脚本引擎对象系统的元数据中枢。

#### GC 相关标志
- `MARK (0x1 << 40)` — 标记-清除 GC 中标记存活对象。
- `ALLOCED (0x2 << 40)` — 对象已在堆上分配，非初始零状态。
- `REFFED (0x10 << 40)` — 对象被外部引用计数器追踪。

#### 生命周期标志
- `STATIC (0x20 << 40)` — 静态对象，GC 永不回收。
- `DIRTY (0x40 << 40)` — 对象已脏（属性被修改过）。
- `FIRST_APPLIED (0x80 << 40)` — 对象已首次应用（apply）。

#### 冻结与权限标志
- `FROZEN (0x100 << 40)` — 完全只读。
- `VALIDATED (0x200 << 40)` — 基类型已针对原型校验。
- `MAP_ADD (0x400 << 40)` — 只允许 map 新增键，不允许修改已有键。
- `VEC_FROZEN (0x800 << 40)` — vec 部分冻结（不可修改 vec 内容）。
- `TYPE_CHECKED (0x1000 << 40)` — 已完成类型检查。
- `NOTPROTO (0x2000 << 40)` — 禁止作为其他对象的原型。

#### 字符串键模式
- `STRING_KEYS (0x4000 << 40)` — 查找时自动在 `LiveId` 与字符串键之间双向转换。用于 JSON 风格的对象访问。

#### 存储模式 (`STORAGE_MASK`, 62-63 位)
- `STORAGE_AUTO` — 自动模式（按需选择）。
- `STORAGE_VEC2` — 仅 `vec` 模式（有序键值对数组）。
- `STORAGE_MAP` — 仅 `map` 模式（HashMap 无序存储）。

#### 引用类型 (`REF_KIND_MASK`, 58-61 位)
存储函数指针、类型索引、POD 类型、ShaderIO 等引用。通过 `ref_data_mask`（低 40 位）存储具体数据：

| 变体 | 用途 |
|------|------|
| `REF_KIND_SCRIPT_FN` | 脚本函数（`ScriptIp` 指令指针） |
| `REF_KIND_NATIVE_FN` | 原生函数（`NativeId` 索引） |
| `REF_KIND_TYPE_INDEX` | 类型索引（`ScriptTypeIndex`） |
| `REF_KIND_POD_TYPE` | POD 类型 |
| `REF_KIND_SHADER_IO` | 着色器 IO 类型 |
| `REF_KIND_APPLY_TRANSFORM` | ApplyTransform 变换 |

#### 关键方法

**`proto_fwd()` / `set_proto_fwd()`**: 从 `PROTO_FWD` 掩码中提取/设置需要在原型链转发时保留的标志位。原型链查询时，新创建的对象需要从原型继承某些元状态，此方法负责这些位的传递。

**`set_type_index()` / `as_type_index()`**: 将类型索引写入标签的低 40 位并设置 `REF_KIND_TYPE_INDEX`。同时标记 `TYPE_CHECKED`。`as_type_index()` 先通过 `is_type_index()` 校验种类，再提取低 40 位构造 `ScriptTypeIndex`。

**`set_fn()` / `as_fn()`**: 存储脚本函数或原生函数。根据 `ScriptFnPtr` 枚举的变体，写入 `ScriptIp` 的 40 位编码或 `NativeId.index`。读取时通过 `REF_KIND_MASK` 分支返回不同枚举变体。

**`is_immutable()`**: 内联检查 `FROZEN | STATIC`，用于快速判断对象是否可修改。这个路径是热点，因为每次属性写入前都要判断。

**`set_pod_type()` / `as_pod_type()`**: 类似 `set_type_index`，但用于 POD 类型引用。注意 `set_pod_type` 不设置 `TYPE_CHECKED` 标志。

**`set_shader_io()` / `as_shader_io()`**: 存储着色器 IO 类型索引。

**`set_dirty()` / `check_and_clear_dirty()`**: `set_dirty` 直接设置 DIRTY 位；`check_and_clear_dirty` 同时执行测试和清除，返回是否曾为脏。典型用途是脏标记检查的原子化操作。

**`freeze` 系列方法**: 一组细粒度冻结策略，每种组合控制 `FREEZE_MASK = FROZEN | VALIDATED | MAP_ADD | VEC_FROZEN` 的不同子集：

| 方法 | 效果 | 语义 |
|------|------|------|
| `freeze()` | 仅 FROZEN | 完全只读 |
| `freeze_type()` | FROZEN + VEC_FROZEN | 类型冻结，不可扩展 vec |
| `freeze_api()` | FROZEN + VALIDATED + VEC_FROZEN | API 冻结 |
| `freeze_module()` | MAP_ADD + VEC_FROZEN + NOTPROTO | 模块冻结，仅可新增 map 键 |
| `freeze_component()` | FROZEN + VALIDATED | 组件冻结 |
| `freeze_shader()` | 全部 4 位 + NOTPROTO | 最严格的着色器冻结 |
| `freeze_ext()` | FROZEN + VALIDATED + MAP_ADD | 扩展冻结，可新增不可修改 |

每种冻结策略先清除 `FREEZE_MASK` 全部位，再设置特定位组合，确保状态原子性。

---

### `ScriptMapTag(u64)` — Map 条目标签
每个 map 键值对的元数据。低 32 位存储**插入顺序**，第 33 位存储**脏标记**。

**`dirty_with_order(order)`**: 构造同时带脏标记和插入序号的标签。新插入的条目立即标记为脏，以便脏追踪系统能识别新增。

**`get_and_clear_dirty()`**: 原子检测并清除脏标记，返回是否为脏。用于增量脏更新检查。

**`order()`**: 提取低 32 位的插入顺序号。用于 `map_iter_ordered` 保持插入顺序遍历。

**`with_order_offset(offset)`**: 在合并 map 时，为每个条目的顺序号加上偏移量（当前 map 长度），确保合并后顺序号全局递增。

---

### `ScriptMapValue`
单个 map 条目：包含 `tag: ScriptMapTag` 和 `value: ScriptValue`。

### `ScriptVecValue`
单个 vec 条目：包含 `key: ScriptValue` 和 `value: ScriptValue`。vec 是有序的键值对列表，键通常是 `LiveId`。

---

### `ScriptObjectData` — 对象核心数据结构
```rust
pub struct ScriptObjectData {
    pub tag: ScriptObjectTag,       // 64位元数据标签
    pub proto: ScriptValue,         // 原型链
    pub map: ScriptObjectMap,       // ValueMap 无序 key-value 存储
    pub vec: Vec<ScriptVecValue>,   // 有序 key-value 数组存储
}
```

这是 Makepad 脚本引擎中所有动态对象的底层数据存储。关键点：一个 `ScriptObject` 同时维护 `map` 和 `vec` 两个独立通道——`map` 提供 O(1) 键查找，`vec` 保持插入顺序、支持按索引访问。

---

## 原型方法 (`add_type_method`) — 核心实现

### `proto` — 获取原型
通过 `vm.bx.heap.proto(sself)` 委托堆管理器查找对象的原型值。

### `push` — 向 vec 追加元素
委托到 `heap.vec_push_vec()`：将调用参数（`args` 中的额外参数）逐个压入目标对象的 `vec`。参数接收通过变长参数列表传递。

### `pop` — 从 vec 弹出尾部
调用 `heap.vec_pop()` 弹出末尾元素并返回。

### `len` — vec 长度
调用 `heap.vec_len()` 返回 `vec.len()`。

### `vec_len` — vec 长度别名
与 `len` 完全相同，但语义上明确是 vec 模式的长度。用于脚本中区分 `vec_len` vs `map_len`。

### `map_len` — map 条目数
调用 `heap.map_len()` 返回 `map.len()`。

### `delete` — 从 map 删除键
实现逻辑：
1. 检查对象是否不可变（`tag.is_immutable()`），若是则返回不可变错误。
2. 调用 `heap.map_delete(sself, &key)` 删除键。
3. 返回值或 NIL。

### `vec_key` — 获取 vec 条目的键
`vec_key(index)` 的实现逻辑：
1. 将索引参数转为 `as_index()`。
2. 调用 `heap.vec_key_value(sself, idx, trap)` 获取键值对。
3. 如果键是 `LiveId`，返回其 `escape()` 值（逃逸为可用的脚本值）；否则直接返回键。

### `gc_id` — 返回 GC 对象索引
返回 `sself.index()`，即该对象在堆数组中的槽位编号。

### `extend` — 批量追加到 vec
调用 `heap.vec_push_vec_of_vec(sself, args, false, trap)`，第二个参数 `false` 表示非 splat 模式：将参数数组的每个元素作为单独条目追加。

### `splat` — 展开追加
与 `extend` 不同之处是第三个参数 `true`，表示 splat 模式：将参数对象视为一个整体，展开其中的所有键值对。

### `freeze` 系列 — 冻结操作
每个 `freeze_xxx` 方法调用堆管理器的对应 `heap.freeze_xxx(sself)`，返回冻结后的对象自身。

### `retain` — 条件保留
`retain(cb)` 的实现逻辑：
1. 遍历 vec 每个索引，获取值。
2. 调用回调函数 `vm.call(fnptr, &[value])`。
3. 如果回调返回 false，调用 `heap.vec_remove(sself, i, trap)` 删除当前元素且**不递增 i**。
4. 如果返回 true，i 递增。
5. 这种"删时留位、不删进位"的策略保证遍历正确。

---

## `ScriptObjectData` 内部方法

### `map_insert(key, value)` — 带脏追踪的插入
核心实现逻辑：
1. 如果 `tag.is_tracked()`，启用脏追踪模式。
2. 尝试 `map.entry(key)` 的 `Occupied` 分支：比较旧值与新值，若不同则设置条目脏标记和对象脏标记。
3. `Vacant` 分支：用 `ScriptMapTag::dirty_with_order(order)` 插入新条目（新条目总是脏的）。
4. 非追踪模式：直接用 `map.insert` 插入，但同样使用 `dirty_with_order` 构造标签。这保证了 map 在任何模式下都有插入顺序号。

### `map_set_if_exist(key, value)` — 仅更新已有键
1. 先用 `entry` API 检查键是否存在（同时处理脏追踪）。
2. 再用 `map.get_mut` 执行无条件更新作为 fallback。
3. 返回 `bool` 表示键是否存在。

### `map_get(key)` — 取值
简单的 `map.get` 查找，返回 `Option<ScriptValue>`。

### `map_get_if_dirty(key)` — 脏值读取
实现逻辑：
1. 追踪模式下，查找条目并调用 `val.tag.get_and_clear_dirty()` 原子检测并清除脏标记，如果曾是脏的则返回值。
2. 非追踪模式退化为普通 `map_get`。

### `map_delete(key)` — 删除
通过 `map.remove(key).map(|v| v.value)` 移除条目并返回其值。

### `map_len()` — 返回 map 条目数
直接委托 `self.map.len()`。

### `map_iter_ret(f)` — 短路迭代
遍历 map，调用闭包 `f(key, value)`。一旦闭包返回 `Some(T)`，立即短路返回；全程无匹配返回 `None`。

### `map_iter(f)` — 无序遍历
遍历 map 的每个条目并调用闭包。

### `map_iter_ordered(f)` — 按插入顺序遍历
实现逻辑：
1. 将 `map.iter()` 的所有条目收集到临时 `Vec`。
2. 按 `val.tag.order()` 排序。
3. 遍历排序后的数组调用闭包。

这是关键方法——因为 `HashMap` 本身无序，而脚本需要保持属性声明顺序时，必须依赖 `ScriptMapTag` 中存储的序数。

### `merge_map_from_other(other)` — map 合并
实现逻辑：
1. 以当前 map 长度作为 `offset`。
2. 遍历目标的 map 条目，用 `with_order_offset(offset)` 调整插入顺序号后插入自身。
3. 这保证了合并后顺序号连续递增。

### `merge_map_from_other_no_overwrite(other)` — 不覆盖合并（splat 操作符）
与 `merge_map_from_other` 区别在于只插入自身不存在的键。用于 `...` (splat) 展开操作符的语义——展开对象不应覆盖已有属性。

### `push_vec_from_other(other)` — vec 批量追加
直接 `extend_from_slice(&other.vec)` 将目标 vec 所有条目追加到当前 vec 末尾。

### `with_proto(proto)` — 设置原型的构造函数
创建一个 `ScriptObjectData`，仅设置 `proto` 字段，其余字段取默认值。用于快速创建带原型的空对象。

### `clear()` — 清空对象
同时清空 `tag`、`map`、`vec`、`proto` 四个字段。包含调试断言确保 map 清空成功。

---

## `ScriptObjectRef` — 引用计数 wrapper

### 设计动机
`ScriptObjectRef` 不是 GC 引用，而是独立的**引用计数句柄**。它通过外部 `roots: HashMap<ScriptObject, usize>` 维护对小对象的引用计数。

### `clone()`
增加引用计数：在 `roots` 中找到对应 `ScriptObject`，将其计数 +1。如果找不到，说明在 GC 回收后仍被引用，打印错误。

### `drop()`
减少引用计数：找到并减 1，减到 0 时从 `roots` 中移除。这种设计允许 `ScriptObjectRef` 在 Rust 侧独立于 GC 管理生命周期，避免对象在 Rust 侧持有期间被 GC 回收。

### `ScriptRefOptionExt`
为 `Option<ScriptObjectRef>` 扩展 `as_object()` 方法，返回 `Option<ScriptObject>`。

---

## 辅助类型

### `ScriptObjectMap`
`ValueMap<ScriptValue, ScriptMapValue>` 的别名。`ValueMap` 是一个自定义的哈希表数据结构，支持 `LiveId` 和字符串键的快速查找。

### `fmt::Display` 实现
`ScriptObjectTag` 的 `Display` 输出形式为 `ObjectType(STORAGE_VEC2|MARK|FROZEN|...)`，列出所有开启的标志位便于调试。
