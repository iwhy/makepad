# `log.rs` — 多平台日志系统

## 用途

`log.rs` 实现了 Makepad 框架的跨平台日志基础设施，提供了统一的日志输出接口，可同时输出到终端、平台原生日志系统（Android logcat、iOS NSLog、OpenHarmony hilog）以及 Studio WebSocket。同时提供了性能分析（profile）和时间格式化辅助宏。

## 日志级别

通过 `makepad_error_log` crate 的 `LogLevel` 枚举定义：
- `Panic` — 崩溃级错误
- `Error` — 错误
- `Warning` — 警告
- `Log` — 常规信息
- `Wait` — 等待/进度信息

前缀映射：`[!]`、`[E]`、`[W]`、`[I]`、`[.]`。

## 核心函数详解

### `Cx::init_log()`

将全局的日志分发函数指针 `LOG_WITH_LEVEL` 设置为 `log_with_level_makepad_platform`。必须在应用初始化时调用一次，使所有日志宏（`log!`、`error!` 等）路由到平台层实现。

### `log_with_level_makepad_platform(...)`

全局日志函数，处理日志的分发路由：

1. **Wasm32 目标**：通过 FFI 调用 `js_console_log` / `js_console_error` 将日志消息输出到浏览器控制台。区分 Error 级别和非 Error 级别以对应 `console.error` 和 `console.log`。

2. **Android 目标**：调用 `__android_log_write`（Android NDK 日志 API），将日志级别映射为 Android 优先级（Error/Panic → 6, Warning → 5, 其余 → 4），tag 固定为 "Makepad"。格式为 `file:line:col - message`。

3. **桌面终端**（非 Android/iOS）：若 Studio WebSocket 未连接，使用 `println!` 输出带前缀的日志。每个消息格式为 `[级别前缀] 文件名:行号:列号 - 消息`。行号和列号从 0 开始，输出时加 1 转换为人类可读的 1-based 格式。

4. **iOS 目标**：通过 FFI 调用 `NSLog`，使用 `str_to_nsstring` 将 Rust 字符串转换为 `NSString`。

5. **OpenHarmony 目标**（`target_env = "ohos"`）：调用 `OH_LOG_Print`（hilog 系统 API），映射日志级别并设置 domain 为 `0x03D00`。使用 `"%{public}s"` 格式说明符。

6. **Studio WebSocket**：若 Studio 连接已启用（`Cx::has_studio_web_socket()`），无论终端输出与否，都通过 `Cx::send_studio_message` 发送 `AppToStudio::LogItem` 消息。每个 `StudioLogItem` 包含完整的文件名、行号范围、列号范围、消息文本和级别。`explanation` 字段始终为 `None`，保留给未来扩展。

## 性能分析工具

### `profile_start() -> ProfileStart / Instant`

跨平台的时间测量起点。非 Wasm32 返回 `std::time::Instant`；Wasm32 返回自定义 `ProfileStart` 结构体，基于 `Cx::time_now()`（高精度时间）计算经过时间。

### `profile_end!` 宏

```rust
profile_end!(instance);
```

计算 `instance.elapsed()` 并以毫秒为单位输出 `Profile time X.XXX ms`。使用 `log::log_with_level` 写入 `LogLevel::Log`。行尾信息通过 `line!() + 4`（`profile_end` 的长度调整）准确定位。

### `profile_end_log!` 宏

```rust
profile_end_log!(instance, "extra info: {}", val);
```

`profile_end` 的扩展版本，允许在性能日志后附加格式化消息。输出格式为 `Profile time X.XXX extra info: val`。

## 格式化辅助宏

### `fmt_over!`

```rust
fmt_over!(dst, "template {}", value);
```

清空目标字符串并写入格式化内容。使用 `std::fmt::Write::write_fmt` 避免分配新的 `String`。不会在失败时 panic（直接 `unwrap`），因为 `String` 的 `write_fmt` 仅在分配失败时才出错。

### `fmt_over_ref!`

```rust
let s = fmt_over_ref!(dst, "template {}", value);
```

与 `fmt_over!` 功能相同，但返回 `&str` 引用而非单元值。方便在表达式上下文中直接使用。

## 设计要点

1. **统一的多平台输出**：通过条件编译（`cfg(target_os)`、`cfg(target_arch)`）在不同平台上激活不同的日志后端，而应用代码只需调用统一的 `log!` 宏。

2. **Studio 协议优先**：无论平台如何，只要 Studio WebSocket 已连接，日志消息始终会发送到 Studio IDE。这使得 Studio 可以集中显示所有平台的日志。

3. **三级日志分发**：先尝试平台原生 API，然后终端输出（作为 Studio 未连接时的后备），最后通过 WebSocket 发送到 IDE，确保日志在任何环境下都能到达。

4. **行号追踪**：所有日志函数接收精确的文件名、起始行/列、结束行/列，便于 Studio 在 IDE 中高亮日志对应的源代码区域。

5. **性能分析集成**：`profile_start`/`profile_end` 的组合提供了零依赖的轻量级性能分析能力，适合在渲染循环等关键路径中使用。
