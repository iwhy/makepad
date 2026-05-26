# JNI 辅助宏

## 概述

`ndk_utils.rs` 是一组减少 JNI 样板代码的宏工具集。它属于独立的辅助层，旨在减少代码中 `(**env).Function.unwrap()` 调用的重复。

## 导出

最后一行 `pub use` 重新导出所有宏，使调用者可以通过 `use crate::ndk_utils::*` 直接使用。

## 宏

### `new_object!`

查找指定类的 `<init>` 方法并调用 `NewObject` JNI 函数。

```rust
new_object!(env, "android/content/Context", "(I)V", arg1)
```

### `call_method!`（核心宏）

调用 JNI 对象的指定方法。**关键优化**：每次展开用到自己独立的 `static AtomicPtr` 缓存 `MethodID`，后续调用跳过 `GetObjectClass`、`CString` 分配和 `GetMethodID`。

```rust
call_method!(CallObjectMethod, env, obj, "getString", "()Ljava/lang/String;")
```

- `$fn`：JNI 调用函数名（如 `CallObjectMethod`、`CallIntMethod`）
- 方法 ID 按调用站点缓存，因为 JNI 方法 ID 在类生命周期内（Android 上即进程生命周期内）稳定

### 类型特定包装

| 宏 | 使用的 JNI 函数 |
|----|----------------|
| `call_object_method!` | `CallObjectMethod` |
| `call_int_method!` | `CallIntMethod` |
| `call_long_method!` | `CallLongMethod` |
| `call_void_method!` | `CallVoidMethod` |
| `call_bool_method!` | `CallBooleanMethod` |
| `call_float_method!` | `CallFloatMethod` |

### 辅助宏

| 宏 | 说明 |
|----|------|
| `get_utf_str!` | 获取 Java 字符串的 UTF-8 内容（返回 `&str`） |
| `new_global_ref!` | 创建 JNI 全局引用 |
| `new_local_ref!` | 创建 JNI 局部引用 |

## 实现说明

- 所有宏展开为 Rust 块表达式，返回 JNI 函数调用结果。
- `call_method!` 的断言（`assert!`）确保 class 和 method ID 不为空，假设 JNI 查找不会失败。
- `get_utf_str!` 调用 `GetStringUTFChars`，调用者需要注意生命周期。
- 此文件不含运行时逻辑，仅在编译时生成代码。
