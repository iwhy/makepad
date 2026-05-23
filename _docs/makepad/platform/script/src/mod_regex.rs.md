# `mod_regex.rs` — 正则表达式模块

## 概述

本文件向 Splash 脚本运行时暴露正则表达式的编译和匹配能力。它注册为 `std.regex` 函数（位于 `mod.std` 模块下），并支持对正则类型（`REDUX_REGEX`）的 `.test()`、`.exec()` 扩展方法以及 `.source`、`.global` 属性 getter。

---

## 脚本 API 详解

### `std.regex(pattern, flags) → script_regex_value`

**注册**: `native.add_method(heap, std, id_lut!(regex), ...)`

**实现逻辑**:
1. 接受两个参数：`pattern`（正则模式字符串）和 `flags`（标志字符串，可省略或传 NIL）。
2. 使用 `vm.bx.heap.string_with` 从脚本值中提取模式字符串。如果非字符串类型，通过 `script_err_type_mismatch!` 报错。
3. 如果 `flags` 非 NIL，同样提取标志字符串。
4. 调用 `vm.bx.heap.new_regex(&pattern_str, &flags_str)`，由底层正则引擎编译模式。
5. 编译成功返回 `ScriptValue` 类型的正则值（底层可能是 `REDUX_REGEX` 变体）。
6. 编译失败调用 `script_err_invalid_args!` 输出错误信息到陷阱。

**标志字符串格式**: 遵循标准标志约定（例如 `"i"` 不区分大小写，`"g"` 全局匹配等），具体由 `vm.bx.heap.new_regex` 实现。

---

### `regex.test(str) → bool`

**注册**: `native.add_type_method(heap, ScriptValueType::REDUX_REGEX, id!(test), ...)`

**实现逻辑**:
1. 提取 `self` 正则引用和参数字符串。
2. 通过 `sself.as_regex()` 获取底层正则指针。如果 `self` 不是正则类型，报 `"test called on non-regex"` 陷阱错误。
3. 在 `heap.string_with` 闭包内获取输入字符串的引用，调用 `re.inner.run(s, &mut [])` 完成匹配检测。
4. `run` 方法的空 `slots` 参数表示只检测是否匹配，不捕获分组。
5. 匹配返回 `ScriptValue::from_bool(true)`，否则返回 `false`。
6. 如果输入值不是字符串类型，报类型错误。

---

### `regex.exec(str) → object {value, index, captures} | nil`

**注册**: `native.add_type_method(heap, ScriptValueType::REDUX_REGEX, id!(exec), ...)`

**实现逻辑**:
1. 提取 `self` 正则引用和输入字符串。
2. 预先从正则对象中获取 `num_captures`（捕获组数），以便分配插槽数组。
3. 分配 `(num_captures + 1) * 2` 个 `Option<usize>` 作为匹配插槽——每个分组对应一个 `(start, end)` 对。
4. 在 `heap.string_with` 闭包内调用 `re.inner.run(s, &mut slots)`。若匹配成功，闭包返回 `Some(input_string_clone)`；否则 `None`。
5. 匹配成功时：
   - 从 `slots[0]`（整体匹配起止）中切割出 `value` 字符串。
   - 创建新对象 `obj = heap.new_with_proto(NIL)`。
   - 设置 `obj.value` 为匹配文本字符串。
   - 设置 `obj.index` 为匹配起始位置（`f64`）。
   - 构建 `captures` 数组：第 0 项为整体匹配，第 1 到 N 项为每个捕获组的匹配文本（或 NIL 表示未捕获）。
   - 设置 `obj.captures` 为捕获数组。
   - 返回对象值。
6. 匹配失败返回 `ScriptValue::NIL`。

---

### Getters: `regex.source`, `regex.global`

**注册**: `native.set_type_getter(ScriptValueType::REDUX_REGEX, ...)`

**实现逻辑**:
- **`source`**: 从 `re.pattern` 克隆原始模式字符串，通过 `heap.new_string_from_str` 分配为脚本字符串返回。
- **`global`**: 返回 `re.flags.global` 布尔标志位，直接转为 `ScriptValue` 布尔值。

---

## 实现要点

- **延迟插槽分配**: `.exec()` 根据 `num_captures` 精确计算需要的插槽数，避免过度分配。
- **零拷贝输入访问**: 使用 `heap.string_with` 回调在借用的生命周期内完成所有匹配操作，避免字符串复制开销。
- **错误隔离**: 所有 `string_with` 的结果通过嵌套的 `Option` 区分"非字符串类型"（外 `Option`）和"正则匹配失败"（内 `Option`）。
- **数组存储切换**: `captures` 数组使用 `ScriptArrayStorage::ScriptValue` 存储模式，通过 `array_mut_mut_self_with` 的闭包方式写入分组数据。
