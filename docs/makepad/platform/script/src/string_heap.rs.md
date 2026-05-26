# `string_heap.rs` — 字符串堆操作

## 文件位置
`platform/script/src/string_heap.rs` (263 行)

## 概述
`string_heap.rs` 在 `ScriptHeap` 上实现了所有与 **ScriptString 字符串值**相关的操作，包括：
- 字符串分配与 intern 驻留（`new_string_with` / `intern_or_store_string`）
- 字符串缓冲区池复用（`temp_string_with` / `strings_reuse`）
- 内联字符串优化（短字符串不分配堆空间）
- 字符串转换（字符串→字节数组、字符数组、类型转换）
- intern 查询（`check_intern_string`）

---

## 方法详解

### 1. `string_mut_self_with<R, F>(&mut self, value, cb) -> Option<R>`
**对字符串值执行可变操作**。如果值是堆字符串（`ScriptString`），则克隆其内容，执行回调 `cb(self, &str)`，并返回结果。如果值是内联字符串（`InlineString`）也同理。这允许在保持脚本引擎所有权不变的情况下，对字符串内容进行可能触发 GC 或修改堆的操作（因为克隆了字符串内容，不持有内部引用）。

### 2. `string_with<R, F>(&self, value, cb) -> Option<R>`
**对字符串值执行只读操作**。与 `string_mut_self_with` 类似，但不克隆字符串——直接借用内部数据。适用于不需要修改堆的纯读取场景。

### 3. `new_string_from_str(&mut self, value: &str) -> ScriptValue`
**从 `&str` 创建字符串**。将 `value` 推入输出缓冲区，然后调用 `new_string_with` 完成 intern 或分配。

### 4. `temp_string_with<R, F>(&mut self, cb) -> R`
**临时字符串分配**。从 `strings_reuse` 缓冲池中取一个已分配的 `String`，执行回调填充它，然后清空并归还到池中。这避免了字符串分配过程中重复分配/释放中间 `String` 对象。

### 5. `new_string_with<F>(&mut self, cb) -> ScriptValue`
**核心字符串分配**。执行步骤：
1. 从 `strings_reuse` 池中获取或新建一个 `String`。
2. 调用回调 `cb` 填充字符串内容。
3. 委托给 `intern_or_store_string` 决定是 intern、复用还是新建堆字符串。

### 6. `intern_or_store_string(&mut self, mut out: String) -> ScriptValue`
**驻留或存储字符串**。核心逻辑：
1. 尝试通过 `ScriptValue::from_inline_string` 将字符串转为内联值。如果字符串长度 ≤ 8 字节（可放入 InlineString），直接返回内联值，并将字符串归还到复用池。
2. 检查 `string_intern` 驻留表：如果该字符串已存在，则复用已有索引，归还字符串到复用池。
3. 尝试从 `strings_free` 复用空闲槽位（GC 已清理并递增了代号），设置 `ScriptStringData`，同时将 `(RcString, ScriptString)` 加入 intern 表。
4. 如果空闲列表为空，将新槽位 push 到 `strings` GenVec 中（起始代号为 0），并加入 intern 表。

### 7. `check_intern_string(&self, value: &str) -> Option<ScriptValue>`
**检查字符串是否已驻留**。先判断能否作为内联字符串，再查询 `string_intern` 表。返回 `ScriptValue` 形式的值。

### 8. `string(&self, ptr: ScriptString) -> &str`
**获取字符串内容**。从 `strings[ptr]` 中的 `ScriptRcString` 取出 `&str`。如果槽位为空（已被回收），返回空字符串 `""`。

### 9. `string_to_bytes_array(&mut self, v: ScriptValue) -> ScriptArray`
**字符串 → U8 字节数组**。处理内联字符串和堆字符串两种情况，将每个字节作为 U8 存入 `ScriptArrayStorage::U8`。用于将字符串转为二进制数组。

### 10. `string_to_chars_array(&mut self, v: ScriptValue) -> ScriptArray`
**字符串 → U32 字符码点数组**。用 `.chars()` 遍历字符串，将每个 Unicode 标量值作为 U32 存入 `ScriptArrayStorage::U32`。

### 11. `cast_to_string(&self, v: ScriptValue, out: &mut String)`
**通用值→字符串格式转换**。依次处理：
- 内联字符串 → 直接写入
- 堆字符串 → 取出内容写入
- f64 / u40 → 数字格式化
- bool → `"true"` / `"false"`
- id → 通过 `LiveId` 的 Display 输出
- nil → 空字符串
- f32 / f16 / u32 / i32 → 数字格式化
- object → `"[ScriptObject]"`
- color → `"#rrggbbaa"` 十六进制格式
- opcode → `"[Opcode]"`
- err → `"[Error:...]"`
- 其他 → `"[Unknown]"`
