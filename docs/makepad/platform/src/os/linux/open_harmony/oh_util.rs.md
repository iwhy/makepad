# oh_util.rs

One-liner (EN): napi utility functions for extracting strings, numbers, properties, and global objects from the OpenHarmony JS environment.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/open_harmony/oh_util.rs` (215 行)
- **核心作用**: 提供一组 napi 工具函数，用于从 OpenHarmony ArkTS 运行时安全地提取 JavaScript 值（字符串、浮点数、对象属性）以及获取 libuv 事件循环和全局上下文。

## 关键函数

### 类型转换

| 函数 | 签名 | 用途 |
|------|------|------|
| `get_value_string` | `(napi_env, napi_value) -> Option<String>` | 将 JS 字符串转为 Rust `String`，先查长度再读取 UTF-8 字节 |
| `get_value_f64` | `(napi_env, napi_value) -> Option<f64>` | 提取 JS 数值为 `f64` |

### 环境/对象查询

| 函数 | 签名 | 用途 |
|------|------|------|
| `get_uv_loop` | `(napi_env) -> Option<*mut uv_loop_s>` | 从 napi 环境获取关联的 libuv 事件循环指针 |
| `get_object_property` | `(napi_env, napi_value, &str) -> Option<napi_value>` | 按名称获取 JS 对象的属性值，检查类型是否有效 |
| `get_global_this` | `(napi_env) -> Option<napi_value>` | 获取 `globalThis` 对象（通过 `napi_get_global` + 属性查询）|
| `get_global_context` | `(napi_env) -> Option<napi_value>` | 获取 ApplicationContext，通过调用 `globalThis.getContext()` |
| `get_files_dir` | `(napi_env) -> Option<String>` | 获取 `context.filesDir` 文件系统路径字符串 |

## 实现细节

- **`get_value_string`**: 两步法 — 先用 `napi_get_value_string_utf8` 查长度，分配缓冲区后再读取。使用 `ManuallyDrop` 防止 Rust 在构造 `Vec` 时释放堆内存。
- **`get_global_context`**: 通过 `globalThis.getContext()` 调用获取 OpenHarmony 的 `ApplicationContext`，用于后续查询 `filesDir` 等系统路径。
- **`get_object_property`**: 在获取属性后额外检查 `napi_typeof`，若值为 `napi_undefined` 则返回 `None`。
- 所有错误路径使用 `crate::error!` 宏记录日志，返回 `None` 表示失败。
- 辅助函数 `value_type_to_string` 将 `napi_valuetype` 枚举转为人类可读字符串，用于调试日志。
