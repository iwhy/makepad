# `mod_std.rs` — Makepad Script 标准库模块

## 文件概述

`mod_std.rs` 定义了 Makepad Script 虚拟机的 `std` 模块，包含基础的运行时函数如断言 (`assert`)、日志输出 (`log`)、标准输出 (`print`/`println`)、范围构造 (`Range`) 以及类型默认值设置 (`set_type_default`)。

所有函数通过 `define_std_module()` 统一注册到虚拟机。

---

## `define_std_module(heap, native)`

主入口函数，创建一个名为 `std` 的脚本模块对象并注册所有方法。

```rust
let std = heap.new_module(id!(std));
```

### 注册流程

1. 在堆上创建 `std` 模块对象
2. 依次调用 `native.add_method()` 注册每个函数
3. 额外创建 `Range` 原型并挂在 `std` 模块下

---

## 注册的函数

### `std.assert(v)`
- **注册名**: `assert`
- **参数**: `v = NIL`（默认 nil）
- **签名**: `script_args!(v = NIL)`
- **内部函数名**: 通过 `id_lut!(assert)` 的 LiveId LUT 方式注册

**实现逻辑**:
1. 通过 `script_value!(vm, args.v)` 获取参数 `v`
2. 调用 `.as_bool()` 尝试转为布尔值
3. 如果值为 `true`，返回 `NIL`（断言通过）
4. 否则调用 `script_err_assert_fail!` 宏触发断言失败陷阱（trap），抛出 `"assertion failed"` 错误信息

**用途**: 脚本中的调试断言，检查某个条件是否为真。

---

### `std.log(what)`
- **注册名**: `log`
- **参数**: `what = NIL`
- **签名**: `script_args_def!(what = NIL)`（使用 LUT 注册参数名）

**实现逻辑**:
1. 获取参数 `what`
2. 调用 `vm.log(what)` — 这将值发送到虚拟机的日志系统，记录到 VM 的日志通道
3. 返回 `NIL`

**用途**: 脚本级别的日志记录，与 VM 日志基础设施集成。

---

### `std.print(what)`
- **注册名**: `print`
- **参数**: `what = NIL`

**实现逻辑**:
1. 获取参数 `what`
2. 尝试通过 `vm.string_with(what, ...)` 获取字符串表示 — 这检查值是否已经是堆内字符串
3. 如果是字符串，用 `print!("{}", str)` 输出到 stdout
4. 如果不是（或者 `string_with` 返回 `None`），使用 `heap.cast_to_string()` 将值转换为临时字符串再输出

**用途**: 脚本端无换行标准输出。

---

### `std.println(what)`
- **注册名**: `println`
- **参数**: `what = NIL`

**实现逻辑**:
1. 获取参数 `what`
2. 首先尝试 `vm.string_with()` 路径 — 如果值是堆内字符串，直接 `println!`
3. 如果失败，进入 `heap.temp_string_with()` 上下文：
   - 调用 `heap.cast_to_string(what, temp)` 将值转为字符串
   - 如果 `temp` 为空（转换失败/值不可转字符串），返回 `true` 标记空
   - 否则 `println!("{}", temp)` 并返回 `false`
4. 如果最后是空字符串，调用 `script_err_unexpected!` 抛出错误：`"println called with empty converted string, value: {:?}"`

**实现细节**: `println` 比 `print` 更严格 — 它检查值是否能被正确转换为字符串，如果不能则报错。这在调试时可以防止静默的空输出。

**用途**: 脚本端带换行标准输出。

---

### `std.Range`
- 这是一个**类型/原型**而非函数
- 通过 `heap.new_with_proto(id!(range).into())` 创建
- 挂载在 `std` 模块下：`heap.set_value_def(std, id!(Range).into(), range.into())`

**用途**: 提供脚本语言中的范围类型。

---

### `Range.step(x)`
- **注册名**: `step`（注册在 `range` 对象上，而非 `std` 模块）
- **参数**: `x = 0.0`
- **签名**: `script_args!(x = 0.0)`

**实现逻辑**:
1. 通过 `script_value!(vm, args.self).as_object()` 获取 `self` 对象（即 Range 实例）
2. 如果 self 是对象，从参数中获取 `x` 的 f64 值
3. 通过 `set_script_value!` 宏设置 `sself.step = x`— 设置 Range 对象的 `step` 属性
4. 返回修改后的 self（链式调用支持）

**用途**: 设置 Range 的步长（例如 `0..10.step(2)`）。

---

### `std.set_type_default(obj)`
- **注册名**: `set_type_default`
- **参数**: `obj = NIL`

**实现逻辑**:
1. 获取参数 `obj` 并尝试 `.as_object()` 转为对象引用
2. 调用 `vm.bx.heap.set_type_default(obj)` — 这将当前对象设置为其类型的默认实例模板
3. 如果设置成功，返回对象本身；否则返回 `NIL`
4. `set_type_default` 的实现将对象标记为 "类型默认值"，后续通过 `Type{...}` 语法创建的新实例将继承这些默认属性值

**用途**: 脚本侧的 `set_type_default` 对应 DSL 中的 `set_type_default() do ...` 语法，用于定义类型默认实例模板。

---

## 设计要点总结

1. **宏辅助**：`script_args!` 创建参数名列表，`script_args_def!` 使用 LUT 版本提高查找性能；`script_value!` 从参数对象提取参数值；`script_err_*!` 系列宏用于错误报告
2. **参数默认值**：每个注册的方法都声明了参数及其默认值（如 `what = NIL`），调用时可以省略可选参数
3. **参数名注册方式**：`id_lut!` 用于大注册量的函数（利用 LiveId 查找表加速），`id!` 用于单次使用
4. **`Range` 作为原型**：`Range` 不是函数而是原型对象，可以实例化并调用其 `step` 方法
5. **字符串输出路径**：`print`/`println` 优先使用 `vm.string_with` 直接获取字符串引用，失败后再 fallback 到 `heap.cast_to_string` 转换
