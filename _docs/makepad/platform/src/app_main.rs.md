# `app_main.rs` — 应用入口与宏定义

此文件是 Makepad 应用的生命周期起点。它定义了 `app_main!` 宏（框架的核心入口宏），以及配套的 `AppMain` trait、跨平台的事件闭包工厂、Studio 连接参数解析工具和 headless 模式参数处理。该文件在编译时通过 `#[cfg]` 条件编译为不同平台（桌面、Android、OHOS、Wasm32）生成对应的入口函数。

---

## `AppMain` Trait

```rust
pub trait AppMain {
    fn script_mod(_vm: &mut ScriptVm) -> ScriptValue { ... }
    fn after_new_from_script(_vm: &mut ScriptVm, _app: &mut Self) { ... }
    fn handle_event(&mut self, cx: &mut Cx, event: &Event);
    fn ui_runner(&self) -> UiRunner<Self> { ... }
}
```

- **`script_mod(vm)`**：注册该应用的脚本模块。默认实现直接 panic，要求具体应用必须实现此方法。它返回一个 `ScriptValue`，该值会被用于后续的应用构造和热重载。
- **`after_new_from_script(vm, app)`**：在应用从脚本构造完成后调用的钩子。默认实现为空。
- **`handle_event(cx, event)`**：核心事件处理方法，每个应用必须实现。这是所有输入事件、窗口事件、定时器、动作等进入应用的主要入口。
- **`ui_runner()`**：返回一个 `UiRunner<Self>`，默认使用 key = 0，假设整个进程只有一个 `AppMain` 实例。

---

## `_app_main_event_closure!` 宏

```rust
macro_rules! _app_main_event_closure {
    ($app:ident) => { ... }
}
```

**职责**：生成一个 `Box<dyn FnMut(&mut Cx, &Event)>` 闭包，该闭包供所有平台入口共享。它捕获两个 `Rc<RefCell<>>`：

1. **`app`**：`Option<App>` 的引用计数包装，存放应用实例。
2. **`app_value`**：`Option<ScriptObjectRef>` 的引用计数包装，存放应用的脚本对象引用（用于 `ScriptReapply`）。

### 闭包内部逻辑

**`Event::Startup`**：
- 调用 `AppMain::script_mod(vm)` 注册脚本模块。
- 将返回值的 object 部分保存为 `ScriptObjectRef`（用于后续热重载时的对象引用）。
- 通过 `ScriptNew::script_from_value` 构造应用实例。
- 调用 `after_new_from_script` 钩子。
- 调用 `cx.start_hot_reload_file_observer_if_requested()` 启动文件热重载观察者（在桌面平台实际生效，其他平台为编译时空操作）。

**`Event::LiveEdit`**：
- 应用接收到 Live 编辑事件时，调用 `script_mod` 重新注册脚本。
- 更新 `app_value` 的脚本对象引用。
- 通过 `ScriptApply::script_apply` 将新脚本应用到现有 `app` 实例上（使用 `Apply::Reload`）。

**`Event::ScriptReapply`**：
- 使用保存的 `app_value` 对象引用，调用 `ScriptApply::script_apply` 以 `Apply::ScriptReapply` 模式应用。
- 这种模式（相对于 `LiveEdit`）不重新注册脚本模块，只重新应用属性值。

**通用事件分发**：
- 在 Startup/LiveEdit/ScriptReapply 处理后，始终调用 `app.handle_event(cx, event)` 将事件分发给应用。

---

## `app_main!` 宏

```rust
macro_rules! app_main {
    ( $ app: ident) => { ... }
}
```

这是 Makepad 应用的标准入口宏。用户在应用代码中通过 `app_main!(MyApp);` 调用，宏会为不同平台展开对应的 `main()` 或导出函数。

### 桌面平台（非 Android、非 OHOS、非 Wasm32）

```rust
#[cfg(not(any(target_os = "android", target_env = "ohos")))]
fn main() { app_main(); }

#[cfg(not(any(target_arch = "wasm32", target_os = "android", target_env = "ohos")))]
pub fn app_main() { ... }
```

**`app_main()` 执行流程**：
1. **初始化日志**：`Cx::init_log()`。
2. **预处理**：`Cx::pre_start()`。如果返回 true（例如在 Studio 中作为动态库运行时），直接返回，不启动独立事件循环。
3. **创建 Cx 上下文**：通过 `_app_main_event_closure!` 生成事件闭包，传给 `Cx::new()` 构造上下文，并用 `Rc<RefCell<>>` 包装。
4. **Studio 连接**：调用 `resolve_studio_http()` 解析 Studio 的 HTTP 地址，通过 `cx.init_websockets()` 建立 WebSocket 连接。
5. **stdin 循环检测**：若命令行包含 `--stdin-loop` 或环境变量 `MAKEPAD_STDIN_LOOP` 为真，设置 `in_makepad_studio = true`。
6. **初始化 OS 后端**：`cx.init_cx_os()` 创建原生窗口和图形上下文。
7. **进入事件循环**：`Cx::event_loop(cx)` 开始主循环。

### Android 平台

```rust
#[cfg(target_os = "android")]
#[no_mangle]
pub unsafe extern "C" fn Java_dev_makepad_android_MakepadNative_activityOnCreate(...) { ... }
```

- Android 入口通过 JNI 导出 `activityOnCreate` 函数，让 Android Activity 在创建时调用。
- 先调用 `apply_studio_env_from_activity` 从 Android Intent 中提取 Studio 连接参数。
- 然后使用 `_app_main_event_closure!` 创建闭包，通过 `Cx::android_entry(activity, || { ... })` 启动 Android 应用。

### OHOS (OpenHarmony) 平台

```rust
#[cfg(target_env = "ohos")]
#[no_mangle]
extern "C" fn ohos_init_app_main(...) { ... }
```

- 通过 NAPI 注册 OHOS 入口，使用 `Cx::ohos_init` 初始化。
- 独立的 `#[napi_derive_ohos::module_exports]` 模块导出函数负责调用 `ohos_init_app_main`。

### Wasm32 平台

```rust
#[cfg(target_arch = "wasm32")]
pub fn app_main() {}

#[export_name = "wasm_create_app"]
pub extern "C" fn create_wasm_app() -> u32 { ... }

#[export_name = "wasm_process_msg"]
pub unsafe extern "C" fn wasm_process_msg(msg_ptr: u32, cx_ptr: u32) -> u32 { ... }

#[export_name = "wasm_return_first_msg"]
pub unsafe extern "C" fn wasm_return_first_msg(cx_ptr: u32) -> u32 { ... }
```

- `app_main()` 在 Wasm32 上是空函数（主函数由 JS 宿主控制）。
- **`create_wasm_app`**：创建 Cx 上下文，初始化 WebSocket 和 OS 后端，将 Cx 的裸指针转为 u32 返回给 JS 宿主。
- **`wasm_process_msg`**：从 u32 指针重建 Cx 引用，调用 `cx.process_to_wasm(msg_ptr)` 处理从 JS 发来的消息。
- **`wasm_return_first_msg`**：获取 `cx.os.from_wasm` 中的第一条消息，释放所有权后返回指针给 JS 宿主。

---

## Studio 连接解析工具

### `resolve_studio_host()`
- 从环境变量 `STUDIO_HOST` 或 `STUDIO` 中获取 Studio 主机地址。
- 通过 `normalize_studio_host` 标准化（补全 `http://` 前缀，去除尾部斜杠等）。

### `resolve_studio_build()`
- 优先从环境变量 `STUDIO_BUILD` 获取 build ID。
- 回退到从 `STUDIO` URL 的查询参数或路径中提取（例如 `http://host/app/42` 或 `...?build=42`）。

### `resolve_studio_crate()`
- 优先从环境变量 `STUDIO_CRATE` 获取 crate 名称。
- 回退到从 `STUDIO` URL 的查询参数中提取（`?crate=xxx`）。

### `resolve_studio_http()`
- 组合 host、build、crate 成完整的 Studio HTTP URL，格式如 `http://127.0.0.1:8001/app?build=77&crate=makepad-example-xr`。
- 只有 host 但无 build 或 crate 时返回空字符串（无意义）。
- 此 URL 被用于建立与 Studio 的 WebSocket 连接，接收热重载更新和调试指令。

### `build_studio_http(host, build, crate)`
- 内部构造函数，组装 URL 的查询参数部分。有值时添加 `build=...` 和 `crate=...`。

### `normalize_studio_host(host)`
- 对用户输入的 host 做标准化：去除空白、尾部斜杠，解析 scheme，提取 host:port 部分，最终输出 `http://host:port` 格式。

### `studio_query_value(studio, key)`
- 从 Studio URL 的查询字符串中提取指定 key 的值。例如从 `...?build=77&crate=xxx` 中提取 `build` 为 `"77"`。

### `extract_studio_build_id(studio)`
- 尝试从 URL 的 `?build=` 查询参数提取 build ID。
- 若不存在，尝试从路径中提取：匹配 `/app/<build_id>` 模式。

### `extract_studio_crate_name(studio)`
- 从 URL 的 `?crate=` 查询参数提取 crate 名称。

### Headless 模式工具

#### `should_run_stdin_loop_from_env()`
- 检测命令行参数中是否有 `--stdin-loop`，或环境变量 `MAKEPAD_STDIN_LOOP` 是否为 1/true/yes/on。
- 在 stdin 循环模式下，应用通过标准输入接收 Studio 协议指令。

#### `should_disable_headless_draw_from_args()`（仅 headless 构建）
- 检测 `--no-draw` 参数，禁用在 headless 模式下的绘制。

#### `headless_draw_cycles_from_args()`（仅 headless 构建）
- 从 `--draws=N` 或 `--draws N` 参数中读取 headless 模式的绘制帧数。
- 返回 `Some(n)`，`n` 至少为 1。

---

## 单元测试

- **`extract_studio_build_id_handles_app_paths`**：验证从路径格式（`/app/42`）和 URL 格式（`http://host/app/77`）中提取 build ID。
- **`extract_studio_build_and_crate_from_query_url`**：验证从查询参数中提取 build 和 crate。
- **`build_studio_http_uses_query_identity`**：验证 `build_studio_http` 正确组装 URL，包括仅有 crate 时仍能生成有效 URL。
