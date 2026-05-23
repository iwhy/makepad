# `arc_string_mut.rs` — 可变引用计数字符串

## 概述

`ArcStringMut` 是一个双态（two-variant）字符串枚举，在同一类型中融合了"脚本堆上的共享引用计数字符串"和"本地拥有的可变字符串"两种能力。它在 Makepad 脚本引擎与 Rust 宿主代码之间的字符串传递场景中充当零拷贝桥接。

## 核心类型

### `enum ArcStringMut`
- **`Rc(ScriptRcString)`**：持有脚本堆中已有字符串的引用计数句柄。从脚本侧读取字符串时直接复用堆上的内存，无需 clone 底层字符数据。
- **`String(String)`**：标准的 Rust 堆分配字符串，支持原地可变操作。

`Default` 实现生成空的 `String` 变体。`Debug` 格式化委托给内部字符串的 `Debug` 实现。

## 方法与实现逻辑

### `as_rc() -> ScriptRcString`
返回一个引用计数句柄。若当前为 `Rc` 变体，直接 clone 内部句柄（O(1) 引用计数递增）；若为 `String` 变体，通过 `ScriptRcString::new` 在脚本堆上分配新拷贝并返回。

### `as_mut() -> &mut String`
获取一个可变的 `&mut String`。若当前为 `Rc` 变体，先将共享字符串解引用为一份全新 `String` 拷贝，再递归调用自身；若已为 `String` 变体，直接返回内部引用。此操作为写时复制（COW）模式——仅在需要修改时才执行深拷贝。

### `as_mut_empty() -> &mut String`
返回一个空的可变字符串引用。对于 `Rc` 变体，直接丢弃共享引用，原地替换为一个全新的空 `String`；对于 `String` 变体，调用 `clear()` 清空缓冲区但保留已分配的容量，避免后续写入时的重复分配。

### `set(v: &str)`
将字符串设置为给定值。`Rc` 变体会替换为包含新内容的 `String`（触发所有权转移）；`String` 变体先 `clear()` 再 `push_str(v)`，复用已有堆分配。

### `as_ref() -> &str`
返回字符串内容的不可变切片。两种变体均直接返回内部 `&str`，`Rc` 变体引用脚本堆，`String` 变体引用本地堆。

## 脚本引擎集成

- **`ScriptHook`**：空实现，标记该类型支持脚本钩子。
- **`ScriptNew`**：`script_new` 生成默认空字符串；`script_type_check` 允许接收任何类字符串（`is_string_like`）的脚本值。
- **`ScriptApply`**：`script_apply` 中，若为重应用（`ScriptReapply`）则直接跳过——因为文本字段应当由命令式 setter（如 `Label::set_text`）控制，DSL 的旧字面量不应覆盖运行时值。仅在 `Reload`（LiveEdit 热重载）时接受新值，通过 `cast_to_string` 将脚本值转为 Rust `String`。
- **`script_to_value`**：优先尝试内联字符串编码（短字符串直接嵌入 `ScriptValue`），失败时在脚本堆上分配新 `String` 并返回其句柄。
