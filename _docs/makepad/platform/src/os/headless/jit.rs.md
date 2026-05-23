# `headless/jit.rs` — JIT 着色器编译系统

## 概述

本文件实现 Makepad 无头模式的 **JIT（Just-In-Time）着色器编译** 系统。它将 Makepad DSL 编写的着色器编译为原生动态库（`.dylib`/`.so`/`.dll`），然后通过 `dlopen`/`dlsym` 在运行时加载函数指针。这是 headless 软件渲染管线的核心环节。

---

## `HeadlessShaderJit` — JIT 编译器管理器

```rust
pub struct HeadlessShaderJit {
    root_dir: PathBuf, // 输出目录根路径
}
```

**`default_jit_root_dir()`** — 确定 JIT 输出根目录：
1. 如果设定了 `CARGO_TARGET_DIR` 环境变量，使用 `$CARGO_TARGET_DIR/makepad-headless-jit`
2. 否则使用当前目录下的 `target/makepad-headless-jit`
3. 可通过 `MAKEPAD_HEADLESS_JIT_DIR` 环境变量覆盖

---

## `compile_and_load` — 核心编译方法

此方法实现了完整的"源码 → 动态库 → 加载"流程：

1. **创建输出目录**: `root_dir / shader_{source_hash:016x}`，以源哈希命名的子目录
2. **写入源文件**: 将着色器 Rust 源码写入 `lib.rs`
3. **调用 rustc**: 执行：
   ```
   rustc --edition=2021 --crate-type=cdylib --crate-name=makepad_headless_shader_{hash} -O lib.rs -o shader_{hash}.so
   ```
   - `--crate-type=cdylib`: 生成 C ABI 兼容的动态库
   - `-O`: 启用优化（速度优先）
4. **加载动态库**: 调用 `HeadlessLoadedModule::load` 通过 `dlopen` 打开
5. **查询版本**: 调用导出函数 `makepad_headless_shader_version()` 获取版本号
6. 返回包含模块句柄和路径的 `HeadlessJitOutput`

编译失败时返回 `Err(String)`，其中包含 `rustc` 的 stderr 输出。

---

## `HeadlessLoadedModule` — 动态库句柄封装

### macOS 实现（`#[cfg(target_os = "macos")]`）

直接使用 POSIX `dlopen`/`dlsym`/`dlclose` 系统调用：

| 方法 | 说明 |
|------|------|
| `load(path)` | `dlopen(path, RTLD_NOW)` — 立即解析所有符号 |
| `shader_version()` | 调用 `makepad_headless_shader_version` 导出函数 |
| `symbol(name)` | `dlsym(handle, name)` → 泛型转换 `transmute_copy` 为函数指针 |
| `Drop::drop` | `dlclose(handle)` — 自动卸载 |

**关键设计**: `symbol::<F>()` 使用 `transmute_copy` 将 `*mut c_void` 转换为任意函数指针类型 `F`。这是动态加载的标准实践，因为 C ABI 中函数指针的表示在所有代码指针之间是一致的。

### 非 macOS 实现

返回错误信息 "headless shader dlopen is only implemented on macOS for now"。这意味着 headless JIT 模式目前仅 macOS 有完整的实际运行时支持，其他平台需要额外的 `libloading` 或平台特定实现。

---

## `dylib_extension()` — 平台动态库扩展名

| 平台 | 扩展名 |
|------|--------|
| Windows | `.dll` |
| macOS | `.dylib` |
| Linux/Unix | `.so` |

---

## 与 `shader.rs` 的关系

- `shader.rs` 中的 `generate_headless_rust_shader_module()` 生成 Rust 源码
- `headless_compile_shaders()` 调用本文件的 `compile_and_load()` 编译该源码
- 编译成功后，通过 `symbol()` 查询 `rcx_size`, `rcx_vary_offset` 等布局信息
- 最终通过 `symbol::<VertexFn/FragmentFn>()` 获取顶点/片段着色器入口函数指针
