# `vec_prims.rs` 源码解读

**路径:** `platform/script/src/vec_prims.rs`
**行数:** 539
**核心职责:** 为常见 Rust 集合类型（`Vec<T>`、`Vec<u8>`、`Vec<ScriptValue>`、`HashMap<K,V>`、`BTreeMap<K,V>`）实现 `ScriptHook`/`ScriptNew`/`ScriptApply` trait，使其能在脚本引擎中作为一等值使用。

---

## `Vec<T>` 脚本集成

### `ScriptHook` 实现（第13行）
空实现，约束 T 必须满足 `ScriptApply + ScriptNew + ScriptDeriveMarker`。

### `ScriptNew`（第14-59行）

- **type_check**（第21-49行）：接受三类输入：
  - **脚本对象**（object）：遍历 vec，递归检查每个元素是否满足 T 的类型
  - **数组**（array）：对 ScriptValue 数组逐元素检查；F32/U32/U16/U8 类型数组自动通过
  - **nil、单元素或 apply_transform**：通过
- **default**：返回 NIL
- **new**：返回 Default::default()
- **proto_build**：返回 NIL

### `ScriptApply`（第60-128行）

- **script_apply**：
  1. 检查 apply_transform
  2. **object**：resize 到 vec 长度，逐个元素 apply
  3. **array**：resize 到数组长度，逐个元素 apply
  4. **nil**：clear
  5. **其他**：clear 后 push 一个 from_value 出来的元素
- **to_value**（第100-128行）：
  1. 创建新数组
  2. 利用 swap 技巧避免重复分配：先从 heap 中取出存储，填充数据，再 swap 回去

---

## `Vec<u8>` 脚本集成

### `ScriptNew`（第131-169行）

- **type_check**：支持 object（元素必须 number）、所有 array 存储类型、string-like、nil、apply_transform
- **default/proto_build**：返回空对象

### `ScriptApply`（第172-249行）

- **script_apply**：
  - **object**：遍历 vec 提取 number 值
  - **array**：各存储类型（ScriptValue/F32/U32/U16/U8）均转为 u8
  - **string（heap）**：取字符串字节
  - **inline string**：as_bytes 写入
  - **nil**：clear
  - **其他**：类型错误
- **to_value**：创建 U8 类型数组存储

---

## `Vec<ScriptValue>` 脚本集成

### `ScriptNew`（第252-269行）

- **type_check**：始终通过（接受任意值）
- **default/proto_build**：返回空对象

### `ScriptApply`（第271-339行）

- **script_apply**：
  - **object**：遍历 vec 收集值
  - **array**：各存储类型均转为 ScriptValue 收集
  - **nil**：clear
  - **其他**：clear 后 push
- **to_value**：创建 ScriptValue 类型数组存储

---

## `HashMap<K, V>` 脚本集成

### `ScriptNew`（第347-379行）

- **type_check**：必须是 object（遍历 map 检查 key/value 类型）或 nil 或 apply_transform
- **default/proto_build**：返回 NIL

### `ScriptApply`（第380-439行）

- **script_apply**：
  - **object**：clear 后遍历 map，逐一调用 `K::script_from_value` / `V::script_from_value` 插入
  - **nil**：clear
  - **其他**：类型错误
- **to_value**：创建新对象，遍历自身插入 map entries。若存在 string-like 的 key 则标记 `set_string_keys`

---

## `BTreeMap<K, V>` 脚本集成

### `ScriptNew`（第447-479行）

- 与 HashMap 相同的检查逻辑，但 K 约束为 `Ord` 而非 `Hash`

### `ScriptApply`（第480-539行）

- 与 HashMap `script_apply` 逻辑完全相同（clear + insert）
- `to_value` 逻辑也相同（创建对象、标记 string_keys）
