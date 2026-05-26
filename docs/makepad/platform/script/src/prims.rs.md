# `prims.rs` 源码解读

**路径:** `platform/script/src/prims.rs`
**行数:** 1050
**核心职责:** 为 Makepad 脚本引擎中的基础 Rust 类型提供 `ScriptHook`/`ScriptNew`/`ScriptApply` trait 实现，使 f32、f64、Vec2d、Vec4f、String、LiveId、Option、HashMap 等类型能在脚本运行时中通过统一的泛型接口进行创建、类型检查、赋值和序列化。

---

## 宏 `script_primitive!`

**路径:** 第13-32行

这是一个批量生成 `ScriptHook` + `ScriptNew` + `ScriptApply` 实现的宏。它接收四种方法（`$new`、`$type_check`、`$apply`、`$to_value`）并展开成完整的 trait 实现块：
- `ScriptHook` 为空实现（表示该类型是叶子类型，没有自定义生命周期钩子）。
- `ScriptNew` 提供 `script_new`、`script_type_name`（从类型名推导 LiveId）、`script_default`（创建实例再转 ScriptValue）和 `script_proto_build`（返回默认值作为原型）。
- `ScriptApply` 提供 `script_type_id`（通过 TypeId::of）和两个核心方法 `script_apply` / `script_to_value`。

---

## 数值类型的 Script 集成

### `f32`（第34-64行）
- **script_new:** 返回 Default::default()（0.0）。
- **type_check:** 接受 number、bool 或存在 apply_transform 的值。
- **script_apply:** 优先从 ScriptValue 中提取 number 赋值；其次 bool 转 1.0/0.0；最后尝试 apply_transform 递归转换。
- **to_value:** 将自身转为 f64 再封装为 ScriptValue。

### `f64`（第66-96行）
- 与 f32 同理，但在 as_number 成功后保留 f64 精度。

### `u64`（第98-128行）
- 同理，但做 `as u64` 截断转换。

### `usize`（第130-160行）
- 同理，做 `as usize` 截断。

### `u32`（第240-270行）
- 同理，做 `as u32` 截断。

### `u16`（第272-302行）
- 同理，做 `as u16` 截断。

---

## 逻辑与引用类型的 Script 集成

### `bool`（第304-324行）
- **script_apply:** 使用 `heap.cast_to_bool` 将任意值转换为布尔语义（truthy/falsy）。
- **to_value:** 直接包装为 ScriptValue::from_bool。

### `String`（第326-363行）
- **script_apply:** 若 `Apply` 为 `ScriptReapply`（由 `cx.request_script_reapply` 触发的堆广播），**直接 return** 避免覆盖运行时 setter 设的值。否则调用 `heap.cast_to_string` 写入。
- **to_value:** 优先使用 inline string 编码（短字符串优化），否则分配堆字符串。

### `&'static str`（第365-389行）
- **script_apply:** 空操作（静态字符串不可变）。
- **to_value:** 同 String，优先 inline 编码。

### `LiveId`（第391-414行）
- **type_check:** 必须是 id 类型。
- **script_apply:** 从 ScriptValue.as_id() 提取。
- **to_value:** 直接 `(*self).into()`。

---

## 堆对象引用的 Script 集成

### `ScriptObjectRef`（第162-184行）
- **type_check:** 必须是 object 类型。
- **script_apply:** 从 ScriptValue 中提取 ScriptObject 并创建新的引用。

### `ScriptFnRef`（第186-214行）
- **type_check:** 必须是 object 且在堆中标记为函数（`heap.is_fn`）。
- **script_apply:** 验证是函数对象后创建引用。

### `ScriptHandleRef`（第216-238行）
- **type_check:** 必须是 handle 类型。
- **script_apply:** 提取句柄并创建引用。

---

## 值类型的直接包装

### `ScriptObject`（第416-438行）
- 直接读写底层 `ScriptObject`。

### `ScriptValue`（第441-462行）
- 恒等转换：类型检查永远通过，apply 直接赋值，to_value 直接返回自身。

---

## `LiveIdMap<K, V>` 的 Script 集成

### `ScriptHook` 实现（第465-477行）
- 空实现，依赖泛型参数约束。

### `ScriptNew`（第479-518行）
- **type_check:** 遍历对象 map，递归检查所有 key 和 value 是否符合 K 和 V 的类型；nil 和 apply_transform 值也接受。
- **default:** 返回 NIL。
- **proto_build:** 返回 NIL。

### `ScriptApply`（第520-578行）
- **script_apply:** 检查 apply_transform；若是 object 则清空自身并逐键值对调用 `K::script_from_value` / `V::script_from_value` 插入；nil 则清空；否则类型错误。
- **to_value:** 创建新 object，遍历自身所有条目，将 key 和 value 分别通过 `script_to_value` 转换后插入 map。

---

## 数学向量/矩阵类型的 Script 集成

### `Vec2d`（第580-656行）
- **type_check:** 接受 number（退化），或 2 维 Vec Pod（支持 Vec2f/Vec2i/Vec2u/Vec2b/Vec2h），或 apply_transform。
- **script_apply:** 若为 number 则 x 和 y 同时赋该值；若为 2D Pod 则按 Pod 类型解析（f32→f64、i32→f64、u32→f64、bool→0/1、f16→f64）；否则尝试 apply_transform。
- **to_value:** 创建 Vec2f Pod 写入。

### `Vec2f`（第658-733行）
- 与 Vec2d 同理，但用 f32 存储而非 f64。

### `Vec3f`（第735-827行）
- **type_check:** 额外接受 color 类型（RGB 通道）。
- **script_apply:** 若是 color 则用 `Vec4f::from_u32` 解包后取前三分量；若是 3D Pod 则按 Vec3f/Vec3i/Vec3u/Vec3b/Vec3h 解析。

### `Vec4f`（第829-926行）
- **type_check:** 同 Vec3f，接受 4D Pod 或 color。
- **script_apply:** color 解码为完整的 RGBA；4D Pod 支持 Vec4f/Vec4i/Vec4u/Vec4b/Vec4h。

### `Mat4f`（第928-990行）
- **type_check:** 接受 4×4 矩阵 Pod 或 number。
- **script_apply:** number 退化为所有 16 个分量设为该值；Pod 必须为 4×4f 类型。
- **to_value:** 创建 4×4 矩阵 Pod 写入。

---

## `Option<T>` 的 Script 集成（第992-1050行）

### `ScriptNew`
- **type_check:** nil 或内部类型可接受。
- **default/proto_build:** 返回 NIL。

### `ScriptApply`（第1015-1050行）
- **script_apply:** 若 `self` 是 `Some`：nil 则置 None，否则递归 apply 到内部值。若 `self` 是 `None`：非 nil 则创建新 T 实例并 apply。
- **to_value:** Some 时委托给内部值，None 时返回 NIL。
