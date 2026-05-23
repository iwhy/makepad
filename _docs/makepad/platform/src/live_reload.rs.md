# `live_reload.rs` — Makepad 热重载引擎

## 用途

`live_reload.rs` 实现了 Makepad 框架的**热重载**基础设施，允许在运行时检测 Rust 源文件中 `script_mod!` 宏的内容变化并实时替换为新的脚本代码，无需重启应用。它通过文件系统观察器（桌面平台）或 Studio WebSocket 协议接收文件变更事件，然后重新解析、验证并应用更新后的 `script_mod!` 块。

## 核心数据结构

### `CxLiveReloadState`
存储运行时热重载状态：
- `pending_files: Vec<PendingLiveChange>` — 暂存待处理的文件变更队列。
- `script_mod_overrides: Rc<RefCell<HashMap<ScriptModKey, String>>>` — 从原始 `script_mod!` 代码到覆盖代码的映射。若某个脚本模板块被修改过，其覆盖代码存储于此。
- `file_observer: Option<DesktopHotReloadWatcher>` — 桌面平台的文件观察器句柄。

### `LiveEditTrigger`
`handle_live_edit` 的返回值枚举：
- `None` — 没有待处理的变更。
- `FileChange` — 文件发生了真实的 DSL 变更，需要完整的重走 UI 树和重置着色器缓存。
- `Manual` — 由 `cx.request_live_edit()` 触发的手动请求（如 iOS 旋转更新安全区域常量），DSL 本身未变，只需重新求值。

### `PendingLiveChange`
```rust
struct PendingLiveChange {
    file_name: String,
    content: String,
}
```
表示一个待处理的文件变更——包括文件路径和完整文本内容。

### `ExtractedScriptMod`
从 Rust 源文件中提取并规范化后的 `script_mod!` 块：
- `code: String` — 规范化后的脚本代码（`#(expr)` 占位符被重编号为 `#(0)`, `#(1)`, ...）。
- `rust_value_count: usize` — 代码中 `#(expr)` 占位符的数量。
- `first_token_line` / `first_token_column` — 第一个非空白 token 的位置（用于解析器的位置追踪）。

### `CompiledScriptModSite`
运行时 VM 中已编译的脚本模板块信息：
- `key: ScriptModKey` — 全局唯一标识。
- `file_name: String` — 原始文件路径。
- `original_code: String` — 编译时记录的原始代码。
- `values: Vec<ScriptValue>` — 原始的 `#(expr)` 求值结果列表。

## 核心函数详解

### `Cx::start_hot_reload_file_observer_if_requested(&mut self)`

桌面平台的入口点。检查 `--hot` 命令行参数，若已设置且尚未启动观察器，则构建监控计划并启动文件观察线程。通过 `channel::<StudioToApp>()` 将文件变更事件转换为 `StudioToApp::LiveChange` 消息发送到 UI 线程，并调用 `SignalToUI::set_ui_signal()` 触发事件循环唤醒。

使用 `LiveReloadLogger` 桥接观察器的日志消息到框架的 `log!` 和 `error!` 宏。若观察器启动失败（如 `inotify` 限制），记录错误但不崩溃。

### `handle_cx_live_edit(cx: &mut Cx) -> LiveEditTrigger`

热重载处理的主路由。先处理文件变更（`handle_cx_live_edit_files`），再处理手动请求（`cx.pending_live_edit_request`）。通过优先级设计确保真实的 DSL 变更优先于手动请求——因为文件变更同时会更新 `script_mod_overrides`，而后续的 `script_mod` 重新求值会读取这些覆盖。

### `handle_cx_live_edit_files(cx: &mut Cx) -> bool`

核心的文件变更处理逻辑：
1. 取出 `pending_files`，使用 `BTreeMap<String, String>` 按文件名去重，保留每个文件最新的变更内容。`BTreeMap` 的有序性确保了跨文件的确定性处理顺序。
2. 对每个变更文件，先克隆当前的 `script_mod_overrides` 作为基线，然后遍历该文件中已编译的 `script_mod!` 块。
3. 对每个块执行**三项前置校验**：
   - 提取出的 `script_mod!` 数量必须与已编译的数量一致（防止增删宏块导致错位）。
   - 提取出的 `#(expr)` 占位符数量必须与原始编译时的数量一致（防止 Rust 插值参数变化）。
   - 新代码必须通过 **tokenizer + parser** 的完整语法验证（防止注入无效脚本语法）。
4. 若新代码与当前生效的代码相同（包括与原始代码相同），跳过以避免不必要的覆盖更新。
5. 通过将所有已验证的新代码原子性地写入 `script_mod_overrides` 完成应用。若任何一步校验失败，整个函数返回 `false`，不应用任何修改。

### `collect_compiled_sites_for_file(script_vm, file_name) -> Vec<CompiledScriptModSite>`

遍历 VM 中所有已编译的脚本体（`code.bodies`），筛选出 `ScriptSource::Mod` 类型的块，然后调用 `resolve_matching_script_mod_file` 判断其源文件路径是否匹配变更文件。使用 `HashSet<ScriptModKey>` 去重（同名但不同来源的块只保留第一个）。结果按行/列排序以保持稳定顺序。

### `validate_extracted_script_mod(script_vm, site, extracted) -> bool`

对新提取的 `script_mod!` 代码体执行快速的语法验证——创建一个临时的 `ScriptTokenizer` 和 `ScriptParser`，运行完整的词法分析和解析。若解析器没有报告错误（`!parser.had_error`），则验证通过。这确保有语法问题的代码永远不会被热加载。

### `resolve_matching_script_mod_file(script_mod, changed_file_name) -> Option<String>

判断某个已编译的 `ScriptMod` 是否来源于变更文件的路径匹配逻辑：
1. 直接路径对比（规范化后比较）。
2. 通过 `resolve_script_mod_file_candidates` 尝试多种候选路径解析（CWD、cargo manifest 祖先目录等）。
3. 路径后缀匹配（至少 3 层组件），用于处理工作区相对路径与绝对路径的对应关系。
4. crate 锚定后缀匹配——对 `src/main.rs` 这种常见短路径，用 crate 目录名锚定以避免跨 crate 误匹配。

### `extract_script_mods_from_rust_file(file_name, source) -> Result<Vec<ExtractedScriptMod>>`

逐字节扫描 Rust 源文件，查找所有 `script_mod! { ... }` 宏调用。扫描过程中正确处理：
- 行注释 `//`、块注释 `/* */`（包括嵌套）
- 字符串字面量（不会误将字符串内的 `script_mod!` 当作宏）
- 原始字符串 `r#"..."#`、字节字符串 `b"..."`、字符字面量

对每个找到的宏体，调用 `normalize_script_mod_body` 进行规范化处理。

### `normalize_script_mod_body(file_name, body, start_pos) -> Result<ExtractedScriptMod>`

对 `script_mod!` 宏体做关键处理：
- 将所有 `#(任意Rust表达式)` 替换为 `#(i)`（`i` 从 0 递增的序号），并在 `rust_value_count` 中计数。
- 注释替换为空白（保留换行以保证行号对齐），因此 `script_mod!` 中不应放注释。
- 末尾追加一个分号 `;` 以确保解析器不会因缺少终结符而出错。
- 追踪第一个非空白 token 的位置用于后续验证。

### `collect_hot_reload_watch_plan(script_vm) -> Option<HotReloadWatchPlan>`

构建文件观察器的监控计划。遍历所有 `ScriptSource::Mod` 块，收集其对应的文件路径。排除 `platform`、`platform/script`、`platform/../draw` 等框架内部 crate（通过 `excluded_hot_reload_manifest_paths`），因为这些是框架本身而非用户代码，修改它们会导致不可预测的行为。对每个文件读取初始内容用于变更检测。

### 辅助函数组

- **`push_unique_candidate`**: 对候选路径去重。
- **`path_has_component_suffix`**: 检查路径是否以给定的组件序列结尾（至少 3 层组件）。
- **`normalized_path_components`**: 将路径拆分为规范化的组件列表（忽略根目录和当前目录）。
- **`skip_line_comment` / `skip_block_comment` / `skip_quoted` / `skip_raw_string` / `char_literal_end`**: 一套完整的 Rust 源码 token 跳越函数，用于在字节级别精确识别各种语法结构。
- **`find_matching_delim`**: 支持嵌套的成对定界符匹配（如 `{...}`、`(...)`）。
- **`skip_non_code_segment`**: 尝试跳过注释、字符串、字符字面量等非代码段。
- **`is_ident_start` / `is_ident_continue`**: 快速判断字节是否为标识符起始/继续字符。

## 设计要点

1. **语法安全前置**：在热重载应用之前，通过完整的 tokenizer + parser 验证确保新代码语法正确。这防止了加载损坏的脚本导致运行时崩溃。

2. **`#(expr)` 占位符重编号**：Rust 编译时 `#(expr)` 被求值替换为具体值，而热重载的脚本代码是 Rust 源码级别。通过将 `#(任意表达式)` 统一重编号为 `#(0)`, `#(1)` 等，热重载系统将新提取的脚本替换到 VM 中时，会复用原始编译时求得的值，从而保证运行时一致性。

3. **文件路径匹配的健壮性**：支持多种路径格式——绝对路径、工作区相对路径、cargo manifest 相对路径、以及带 crate 锚定的后缀匹配——以适应不同平台和构建配置下的文件路径差异。

4. **框架自保护**：通过 `excluded_hot_reload_manifest_paths` 主动屏蔽框架自身的 crate 目录，防止开发者误修改框架代码触发不可预测的热重载行为。

5. **`LiveEditTrigger` 的可区分语义**：通过区分 `FileChange` 和 `Manual`，调用者可以精确控制后续的恢复操作——文件变更需要完全重走 UI 树和重置着色器缓存，而手动请求只需重新求值脚本中的表达式。
