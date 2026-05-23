# lib.rs — Splash VM 模块导出与宏定义

## 概述

`lib.rs` 是 Splash 脚本引擎 crate 的根入口文件。它定义了 crate 的公共 API 导出、模块声明、以及核心宏 `script_eval!`。

**总行数**: 92 行

---

## 外部 crate 再导出 (行 1-7)

```rust
pub use makepad_error_log;
pub use makepad_live_id;
pub use makepad_live_id::makepad_live_id_macros;
pub use makepad_math;
pub use makepad_math::makepad_micro_serde;
pub use makepad_regex;
pub use makepad_script_derive;
```

这些 `pub use` 将依赖 crate 中最重要的类型和宏重新导出到 `makepad_script` 命名空间，使外部使用者不需要直接依赖这些子 crate：

| 再导出 | 提供的内容 |
|--------|-----------|
| `makepad_error_log` | 日志记录基础设施（`log!`, `log_with_level` 等） |
| `makepad_live_id` | LiveId 类型和 `id!`/`ids!` 宏（基于 xxhash 的 64 位标识符） |
| `makepad_live_id_macros` | LiveId 宏的额外功能 |
| `makepad_math` | 数学库类型（Vec2, Vec4, Mat4 等） |
| `makepad_micro_serde` | 微序列化/反序列化工具 |
| `makepad_regex` | 正则表达式引擎 |
| `makepad_script_derive` | 派生宏（`Script`, `ScriptHook` 等） |

---

## `script_eval!` 宏 (行 9-14)

```rust
#[macro_export]
macro_rules! script_eval {
    ($vm:expr, { $($tt:tt)* } $(,)?) => {{
        ($vm).with_vm(|vm|{let b = $crate::script! { $($tt)* };vm.eval(b)})
    }};
}
```

**功能**: 在 `ScriptVm` 上下文中编译并执行一段内联脚本代码。

**执行流程**:
1. 通过 `($vm).with_vm(|vm| { ... })` 获取可变引用
2. 调用 `$crate::script! { $($tt)* }` 宏，将 inline DSL 代码转换为 `ScriptMod` 结构体
3. 调用 `vm.eval(b)` 执行编译后的脚本

**使用示例**:
```rust
script_eval!(vm, {
    let x = 42
    let y = x + 1
});
```

这个宏是 Makepad 脚本 DSL 的运行时入口点之一，适合在 Rust 宿主中执行小段脚本逻辑。

---

## 模块声明 (行 16-76)

crate 包含 60 个模块，按功能分组：

### 核心运行时 (6 个)
| 模块 | 用途 |
|------|------|
| `heap` | `ScriptHeap` — 堆分配器（对象、数组、字符串、POD、正则） |
| `thread` | `ScriptThreads` — 线程调度器（call stacks, scope stacks, loop stacks） |
| `trap` | `ScriptTrap` — 控制流追踪（goto, return, pause, bail, try/catch） |
| `value` | `ScriptValue` — NaN-boxed 64 位值系统 |
| `value_map` | `ScriptObjectMap` — 对象 map 实现（HashMap-like） |
| `vm` | `ScriptVm` — 解释器主循环、模块管理、函数调用 |

### 编译系统 (4 个)
| 模块 | 用途 |
|------|------|
| `tokenizer` | `ScriptTokenizer` — 词法分析器 |
| `parser` | `ScriptParser` — 语法分析器，生成操作码 |
| `opcode` | `Opcode` / `OpcodeArgs` — 操作码定义 |
| `opcodes` | 主操作码调度 |

### 操作码分组 (7 个)
| 模块 | 用途 |
|------|------|
| `opcodes_assign` | 赋值操作码（`=`, `+:=` 等） |
| `opcodes_calls` | 函数调用操作码（`call`, `prop_call` 等） |
| `opcodes_control` | 控制流操作码（`if`, `while`, `for`, `match` 等） |
| `opcodes_loops` | 循环操作码（`break`, `continue`） |
| `opcodes_ops` | 运算符操作码（`+`, `-`, `*`, `/`, `==` 等） |
| `opcodes_vars` | 变量操作码（`let`, `var`, 作用域管理） |

### 类型系统 (6 个)
| 模块 | 用途 |
|------|------|
| `array` / `array_heap` | ScriptArray 类型定义和堆存储 |
| `object` / `object_heap` | ScriptObject 类型定义和堆存储 |
| `string` | ScriptString 类型定义和内联字符串 |
| `string_heap` | 字符串堆（内联 + GC 追踪的字符串池） |

### 特殊类型 (5 个)
| 模块 | 用途 |
|------|------|
| `function` | `ScriptFnPtr` — 函数指针（原生 / 脚本） |
| `handle` | `ScriptHandle` / `ScriptHandleGc` — 外部资源引用 |
| `pod` / `pod_heap` | 固定内存布局的 POD 类型系统 |
| `gc` | 垃圾回收器（mark & sweep） |
| `numeric` | 数值转换工具 |

### 原生模块 (7 个)
| 模块 | 用途 |
|------|------|
| `native` | 原生函数注册表（`ScriptNative`） |
| `mod_std` | 标准库（print, type, len, panic 等） |
| `mod_math` | 数学库（sin, cos, sqrt, min, max 等） |
| `mod_regex` | 正则表达式操作 |
| `mod_html` | HTML 渲染引擎接口 |
| `mod_shader` | GPU 着色器编译和链接 |
| `mod_gc` | GC 控制（gc.collect, gc.enable 等） |

### 着色器后端 (6 个)
| 模块 | 用途 |
|------|------|
| `shader` | 着色器抽象层 |
| `shader_backend` | 后端抽象 |
| `shader_builtins` | 内建着色器函数 |
| `shader_calls` | 着色器函数调用代码生成 |
| `shader_control` | 着色器控制流代码生成 |
| `shader_glsl` / `shader_hlsl` / `shader_metal` / `shader_wgsl` | GLSL/HLSL/Metal/WGSL 后端 |
| `shader_ops` | 着色器运算符代码生成 |
| `shader_output` | 着色器输出代码生成 |
| `shader_tables` | 着色器绑定表 |
| `shader_vars` | 着色器变量 |

### 工具模块 (6 个)
| 模块 | 用途 |
|------|------|
| `apply` | `script_apply_eval!` 宏支持 |
| `colorhex` | 颜色十六进制解析 |
| `gen_index` | GenVec 索引生成器 |
| `json` | JSON 解析和序列化 |
| `prims` | 基本几何类型 |
| `suggest` | 类型建议（错误消息改进） |
| `test` | 测试辅助工具 |
| `traits` | 共享 trait 定义 |
| `vec_prims` | 向量基元类型 |

---

## 公共 API 再导出 (行 78-92)

```rust
pub use apply::*;
pub use array::*;
pub use function::*;
pub use gc::*;
pub use handle::*;
pub use heap::*;
pub use makepad_live_id::*;
pub use makepad_script_derive::*;
pub use object::*;
pub use string::*;
pub use thread::*;
pub use traits::*;
pub use trap::*;
pub use value::*;
pub use vm::*;
```

这些 `pub use` 将最常用的模块中的所有公共类型提升到 crate 根级别。外部用户只需 `use makepad_script::*` 即可访问：

- `value::*` → `ScriptValue`, `ScriptObject`, `ScriptArray`, `ScriptString`, `NIL`, `TRUE`, `FALSE` 等
- `vm::*` → `ScriptVm`, `ScriptVmBase`, `ScriptCode`, `ScriptBody`, `ScriptMod` 等
- `heap::*` → `ScriptHeap`, `ScriptObjectRef` 等
- `trap::*` → `ScriptTrap`, `ScriptTrapOn` 等
- `thread::*` → `ScriptThreads`, `ScriptThread`, `CallFrame` 等
- `object::*` → `ScriptObjectMap`, `ScriptVecValue` 等
- `array::*` → `ScriptArrayMap` 等
- `function::*` → `ScriptFnPtr` 等
- `gc::*` → `GcControl` 等
- `handle::*` → `ScriptHandle`, `ScriptHandleType` 等
- `string::*` → `ScriptStringMap` 等
- `traits::*` → 通用 trait
- `apply::*` → `script_apply_eval!` 宏支持

---

## 文件信息
- **路径**: `platform/script/src/lib.rs`
- **行数**: 92
- **模块总数**: 60
- **宏导出**: `script_eval!`
