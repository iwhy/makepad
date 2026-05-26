# `mod_gc.rs` — GC 控制接口模块

## 概述

本文件向 Splash 脚本运行时暴露垃圾回收（GC）的显式控制能力。模块注册在 `mod.gc` 命名空间下，提供四个原生方法：`set_static`、`run`、`run_status` 和 `dump_tag`。

---

## 数据结构和方法详解

### `define_gc_module(heap, native)`

使用 `heap.new_module(id!(gc))` 创建 `gc` 模块对象，然后通过 `native.add_method` 依次注册四个方法。

---

### `gc.set_static(value) → value`

**注册方式**: `native.add_method(heap, gc, id_lut!(set_static), ...)`

**实现逻辑**:
1. 接收一个脚本值作为参数，使用 `script_value!` 宏从 `args.value` 中解包。
2. 调用 `vm.bx.heap.set_static(value)` 将该值的 GC 标记设为"静态"——这意味着 GC 收集器在后续遍历中会跳过此值及其引用的所有对象，将其视为永久根。
3. 原值作为返回值传递回脚本，支持链式调用。

**用途**: 用于保护某些关键对象（如全局配置、主题对象）不被 GC 回收，防止因引用丢失导致的意外释放。

---

### `gc.run() → NIL`

**注册方式**: `native.add_method(heap, gc, id_lut!(run), script_args!(), ...)`

**实现逻辑**:
1. 无参数调用。
2. 直接调用 `vm.gc()` 触发一次完整的垃圾回收遍历。
3. 返回 `ScriptValue::NIL`。

**用途**: 脚本主动触发 GC，在内存敏感的大操作之后手动回收不再使用的对象。

---

### `gc.run_status() → NIL`

**注册方式**: `native.add_method(heap, gc, id_lut!(run_status), script_args!(), ...)`

**实现逻辑**:
1. 与 `run()` 类似，但调用 `vm.gc_with_status()`。
2. `gc_with_status` 在回收后会输出详细的 GC 状态信息（回收对象数、存活对象数等）到日志系统，用于调试和分析内存使用情况。
3. 返回 `ScriptValue::NIL`。

**用途**: 调试时替代 `run()`，用于观察 GC 行为和验证内存泄漏。

---

### `gc.dump_tag(value) → value`

**注册方式**: `native.add_method(heap, gc, id_lut!(dump_tag), ...)`

**实现逻辑**:
1. 接收一个脚本值作为参数。
2. 将值解释为对象引用（`value.as_object()`），如果失败则直接返回原值。
3. 从堆中取出 `ScriptObject`，获取其 `tag`（类型标签）。
4. 提取以下调试信息并通过 `log!` 宏输出：
   - `obj.index`: 对象的堆索引
   - `type_index`: 标签中的类型索引（如果存在）
   - `is_static`: 该对象是否为静态（不可回收）标记
   - `proto`: 对象的原型对象索引
   - `proto_type_index`: 原型对象的类型索引（如存在）
   - `props`: 如果标签包含类型索引，则从 `type_check` 表中读取该类型的所有属性名，以逗号连接成字符串
5. 返回原值。

**用途**: 用于调试脚本引擎本身——检查某个对象的类型、原型链和属性信息，帮助理解对象的结构和 GC 状态。

---

## 设计要点

- **安全设计**: `dump_tag` 不修改任何状态，只读访问，适合在线上调试环境安全使用。
- **链式友好**: `set_static` 返回原值，可以在复杂表达式中嵌入使用。
- **延迟性**: `run` 和 `run_status` 不尝试预测何时执行 GC，完全由脚本显式控制，避免了对 Splash 精细 GC 策略的干扰。
