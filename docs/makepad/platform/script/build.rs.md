# `build.rs` 源码解读

**路径:** `platform/script/build.rs`
**行数:** 1
**核心职责:** Cargo 构建脚本 — 当前为空实现，不执行任何编译时操作

---

## 函数

### `fn main() {}`
**第 1 行**

Cargo 约定的构建入口函数。`platform/script` crate 是一个纯 proc-macro 库，不依赖编译时代的代码生成、代码签入、链接配置或平台检测等构建脚本功能。该文件保持为空函数体，表明本 crate 完全依赖 Rust 编译器的标准编译流程和 proc-macro 机制，无需额外的构建步骤。

**关于 build.rs 的存在理由：** 在某些情况下，即使不需要构建逻辑也需要保留 `build.rs` 以防止 Cargo 对整个 package 施加某些限制（例如包含 `proc-macro = true` 的 crate 如果同时是 workspace 的一部分，某些 workspace 配置可能需要 `build.rs` 存在）。这里保持最小实现，方便未来扩展（如代码生成、资源嵌入等）。
