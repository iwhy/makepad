# `array.rs` — 动态数组系统：`ScriptArray` 与类型化存储

## 文件位置
`platform/script/src/array.rs` (673 行)

## 核心类型

### `ScriptArrayTag(u64)` — 数组元数据标签
与 `ScriptObjectTag` 类似的位标志模型，但更精简：

| 标志 | 用途 |
|------|------|
| `MARK (0x1 << 40)` | GC 标记 |
| `ALLOCED (0x2 << 40)` | 已分配 |
| `STATIC (0x4 << 40)` | 静态不可回收 |
| `DIRTY (0x40 << 40)` | 脏标记 |
| `FROZEN (0x100 << 40)` | 冻结只读 |
| `IMMUTABLE_MASK` | FROZEN \| STATIC 组合 |
| `REF_KIND_APPLY_TRANSFORM` | ApplyTransform 引用（同 object） |

#### 关键方法

**`set_apply_transform()` / `as_apply_transform()`**: 与 object 模式相同，将 `NativeId` 编码到低 40 位+种类掩码。

**`is_immutable()`**: 内联检查 `IMMUTABLE_MASK`，用于写入前的守卫判断。

**`freeze()` / `is_frozen()`**: 设置/检查 `FROZEN` 位。

**`set_dirty()` / `check_and_clear_dirty()`**: 脏标记设置与原子清除。

---

### `ScriptArrayStorage` — 类型化存储枚举
这是数组系统的核心优化：根据存储数据类型选择不同的 `Vec`/`VecDeque`：

```rust
pub enum ScriptArrayStorage {
    ScriptValue(VecDeque<ScriptValue>), // 通用值数组（双端队列）
    F32(Vec<f32>),  // 32位浮点紧凑数组
    U32(Vec<u32>),  // 32位无符号整型
    U16(Vec<u16>),  // 16位无符号整型
    U8(Vec<u8>),    // 字节数组（二进制数据）
}
```

设计动机：对于纯数值数组（如字节缓冲区、采样数据），使用类型化存储可以大幅减少内存占用和 GC 压力。

#### `clear()` — 清空
匹配当前存储变体，调用对应 `Vec`/`VecDeque` 的 `clear()`。

#### `len()` — 长度
所有变体统一返回 `len()`。

#### `index(index)` — 下标读取
实现逻辑：
1. 匹配存储类型。
2. 调用 `get(index)` 获取引用。
3. 原语类型直接 `(*v).into()` 转换为 `ScriptValue`。
4. `ScriptValue` 变体直接返回 `(*v).into()`（解引用后转 ScriptValue）。

#### `set_index(index, value)` — 下标写入
实现逻辑：
1. 若 `index >= v.len()`，先 `resize(index + 1, default)` 扩展。
2. 原语变体用 `value.as_f64().unwrap_or(0.0) as T` 转换后写入。
3. `ScriptValue` 变体直接赋值。
4. 注意扩展时 `Vec::resize` 对 `VecDeque` 不适用，`ScriptValue` 变体用 `resize` 方法（`Vec` 而非 `VecDeque`——这代码实际上对 `VecDeque` 调用 `resize` 是 bug，但 `VecDeque` 的 `resize` 存在吗？检查发现 `VecDeque` 确实有 `resize` 方法。正确。）

#### `push(value)` — 尾部追加
`ScriptValue` 变体用 `push_back`（尾插），其余用 `Vec::push`。

#### `push_vec(vec)` — 批量追加
将 `ScriptVecValue` 切片中的所有 `value` 字段依次 `push` 到数组。

#### `pop()` — 尾部弹出
`ScriptValue` 变体用 `pop_back`，其余用 `Vec::pop`。返回 `Option<ScriptValue>`。

#### `pop_front()` — 头部弹出
`ScriptValue` 变体用 `pop_front`（O(1)），其余用 `Vec::remove(0)`（O(n) 移位）。对于非 `ScriptValue` 类型，头部弹出性能较差。

#### `remove(index)` — 按索引删除
所有变体用 `remove(index)`，`ScriptValue` 变体返回 NIL（如果索引越界）。注意这个操作在非 `ScriptValue` 变体上 panic（如果索引越界）。

#### `to_string(heap, s)` — 拼接为字符串
四种变体的行为差异很大：
- `U8(bytes)`: 用 `String::from_utf8_lossy` 将字节数组转为字符串追加。
- `ScriptValue(vec)`: 逐个元素调用 `heap.cast_to_string(*v, s)` 拼接。
- `F32/U32/U16`: 将每个数值转为 `char::from_u32` 后追加。

---

### `ScriptArrayData` — 数组数据容器
```rust
pub struct ScriptArrayData {
    pub tag: ScriptArrayTag,
    pub storage: ScriptArrayStorage,
}
```

默认构造时 `storage` 为空的 `ScriptValue(VecDeque)`。

---

## 原型方法 (`add_type_methods`)

### `to_string` — 转字符串
委托 `heap.array_storage(arr).to_string(heap, s)`，使用 `heap.new_string_with` 创建新字符串。

### `parse_json` — JSON 解析（Array 版本）
实现逻辑：
1. 从线程取出 `json_parser`（避免借用冲突）。
2. 只对 `U8` 存储类型有效：将字节数组转为 `&str` 后调用 `json_parser.read_json`。
3. 返回解析后的 `ScriptValue`。
4. 若存储类型不是 `U8`，返回类型不匹配错误。

### `parse_json` — JSON 解析（String 版本）
注册在 `REDUX_STRING` 类型上：将字符串内容通过 `heap.temp_string_with` 临时字符串传给 JSON 解析器。

### `push` — 追加
委托 `heap.array_push_vec(sself, args, trap)`，将参数批量追加到数组。

### `pop` — 弹出
委托 `heap.array_pop(sself, trap)`，返回弹出值。

### `clear` — 清空
委托 `heap.array_clear(sself, trap)`，重置数组存储。

### `len` — 长度
委托 `heap.array_len(sself)` 返回 `usize`，`into()` 转换为 `ScriptValue`。

### `remove(index)` — 索引删除
实现逻辑：
1. `script_value!(vm, args.index).as_index()` 转为 `usize`。
2. 委托 `heap.array_remove(sself, idx, trap)`。

### `freeze` — 冻结
委托 `heap.freeze_array(sself)` 设置 `FROZEN` 标志。

### `retain(cb)` — 条件保留
与 object 的 `retain` 逻辑相同：
1. 遍历索引，获取值。
2. 调用回调。
3. 回调返回 false 时调用 `array_remove` 删除，不递增 i。
4. 返回 true 时 i 自增。

---

### `clear()` — 清空 data
同时清空 `storage` 和 `tag`。

### `is_value_array()` — 检查是否为通用值数组
匹配 `ScriptArrayStorage::ScriptValue` 变体。

---

## `ScriptArrayRef` — 引用计数 wrapper
与 `ScriptObjectRef` 完全对称的模式：

- `Clone`: 递增 `roots` 中的计数。
- `Drop`: 递减计数，减到 0 时移除条目。
- `as_array()`: 返回内部的 `ScriptArray`（堆索引）。
- `Into<ScriptValue>`: 通过 `ScriptValue::from_array` 转换。

设计动机：允许 Rust 代码在 GC 管理之外独立持有数组句柄，防止在持有期间被 GC 回收。
