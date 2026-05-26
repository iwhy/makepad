# `scratch.rs` 源码解读

**路径:** `platform/script/src/scratch.rs`
**行数:** 26
**核心职责:** 这是一个草稿/沙盒文件，包含脚本引擎的示例代码片段，并非正式的源代码模块。展示了 fib 函数、App 定义、HTTP 服务器等语法设想。

---

## 内容概要

**第4-5行**: 递归计算斐波那契数的 lambda 表达式 `fib = |n| if n <= 1 n else fib(n - 1) + fib(n - 2)` 及 `~fib(34)` 的日志调用。

**第7-22行**: 一个 `MyApp` 的 DSL 定义，包含 `AppBar`、`TextInput`（带 `on_enter` 事件，触发 AI 流式对话）、`Label`（通过 `<=>` 双向绑定流数据）。

**第24-26行**: HTTP 服务器示例 `http.server(8080, { on_request: \|req, res\| res.write(200, "Working") })`，展示脚本语言对网络 I/O 的设想。

> 注意：此文件不包含任何可编译的 Rust 代码、函数定义或模块声明，仅作为语言设计的草稿笔记。
