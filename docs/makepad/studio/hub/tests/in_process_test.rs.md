# in_process_test.rs

## 概述

这是 Hub 测试套件中最大的文件（2904 行），覆盖了 in-process 模式下 Hub 后端的大部分功能：文件树加载、Cargo 构建生命周期、终端（PTY）管理、Splash 脚本执行、RunItem 调度、文件监控、搜索和文本文件操作。

## 测试基础设施

- `wait_for_message(connection, timeout, matcher)`: 轮询 `HubConnection.recv_timeout` 直到匹配特定 `HubToClient` 消息。
- `drain_messages(connection, duration)`: 收集一段时间内的所有消息（用于验证某类消息 *没有* 出现）。
- `framebuffer_to_text(frame)`: 将 `TerminalFramebuffer` 的 cells 数组解码为可读文本（每个 cell 7 字节：4 字节 codepoint + 3 字节属性）。
- `wait_for_terminal_frame_contains(connection, path, needle, timeout)`: 等待终端 framebuffer 中某行包含指定文本。
- `wait_for_terminal_frame_where(connection, path, timeout, predicate)`: 更灵活的 framebuffer 匹配。
- `wait_for_terminal_shell_ready(connection, path, timeout)`: 通过打印一个标记字符串并等待 framebuffer 中出现该标记来确认 shell 就绪。
- `request_terminal_viewport`: 发送 `TerminalViewportRequest` 设置终端视口大小和滚动位置。
- `terminal_test_shell_env`: 创建自定义 shell 脚本，设置 `SHELL` 环境变量以确保测试环境一致性（Unix only）。

---

## 测试场景分组

### 1. 构建生命周期 (lines 195-277)

#### `in_process_connection_roundtrip_and_cargo_build_lifecycle`

- **场景**: 完整测试 in-process 模式下的构建生命周期。
- **流程**:
  1. 创建临时 Cargo 项目（含有 `src/lib.rs`）。
  2. `StudioHub::start_in_process` 启动后端。
  3. 发送 `LoadFileTree` → 收到 `FileTree`（验证 `repo/src/lib.rs` 存在）。
  4. 发送 `Cargo { args: ["--version"] }` → 收到 `BuildStarted`。
  5. 收到 `BuildStopped`（构建完成）。
  6. `QueryLogs` 查询日志 → 收到 `QueryLogResults`（`done: true`，entries 非空）。
- **验证重点**: In-process 模式下构建的 start → output → stop 完整链路，日志检索功能。

---

### 2. 终端/PTY 测试 (lines 279-1092)

所有终端测试仅在 `cfg(any(target_os = "macos", target_os = "linux"))` 下运行。

#### `terminal_large_paste_keeps_session_alive` [ignored]

- **场景**: 512KB 大粘贴不会导致终端崩溃或断开。
- **流程**: `stty -echo` 关闭回显 → `cat > /dev/null` → 发送 512KB 数据 → Ctrl-C 中断 → `echo __paste_ok__` → 验证标记出现。
- **注意**: 依赖于交互式 shell 启动和 PTY 缓冲区行为，标记为 `#[ignore]`。

#### `terminal_bell_sets_title_badge_until_next_input`

- **场景**: 终端响铃字符（`\a`）触发标题徽章（`@ bell.term`），下一次输入后清除。
- **流程**: 打开终端 → 发送 `printf '\a'` → 收到 `TerminalTitle { title: "@ bell.term" }` → 发送 `:\n` → 收到 `TerminalTitle { title: "bell.term" }`（徽章清除）。
- **验证重点**: 标题徽章机制——`@` 前缀表示有未读输出（bell）。

#### `terminal_resize_delivers_sigwinch_with_updated_stty_size` [ignored]

- **场景**: 终端尺寸改变后通过 SIGWINCH 信号通知 shell，`stty size` 应反映新尺寸。
- **流程**: 启动自定义脚本（trap WINCH 信号报告 `stty size`）→ 10 行 `__SIZE__:10 80` → resize 到 20 行 → `__SIZE__:20 80` → resize 到 20 列 120 行 → `__SIZE__:20 120`。
- **验证重点**: PTY resize → SIGWINCH → `stty` 更新 → framebuffer 内容反映新尺寸。
- **注意**: 依赖 PTY 的 SIGWINCH 时序。

#### `terminal_bash_prompt_sticks_to_bottom_after_grow_resize` [ignored]

- **场景**: 增加视口行数时，bash 提示符应保持在底部。
- **流程**: 打开 bash → 生成 180 行滚动历史 → 10 行视口时 cursor_row 应为 9 → resize 到 15 行 → cursor_row 应为 14，top_row 不增加。
- **验证重点**: 底部锚定行为——视口扩展时提示符不丢失。

#### `terminal_bash_grow_resize_clamps_to_top_when_history_is_insufficient`

- **场景**: 滚动历史不足时，resize 不应强制将光标推到新底部行。
- **流程**: 打开新 shell（几乎无历史）→ 10 行视口 `top_row == 0` → resize 到 15 行 → `top_row == 0`，`cursor_row < 14`。
- **验证重点**: 顶部箝位逻辑——历史不够时顶部不能变为负值。

#### `terminal_codex_prompt_sticks_to_bottom_after_resize`

- **场景**: Codex（终端编辑器）的提示符在 resize 后应保持在底部。
- **流程**: 生成滚动历史 → 启动 `codex` → 10 行视口 → resize 到 15 行 → 最后一行非空（提示符可见）。
- **注意**: 需要 `codex` 二进制存在于 PATH。

#### `terminal_codex_fast_resize_roundtrip_preserves_top_and_bottom_rows` [ignored]

- **场景**: 快速 resize 循环（21-40 行，5 个 cycle）后，视口内容应与 baseline 完全一致。
- **验证**: 顶部行、分隔行、底部行和完整 framebuffer 文本在每个 cycle 后均保持不变。
- **注意**: 代码编辑器 UI 的不确定性，标记为 `#[ignore]`。

#### `terminal_codex_fast_vs_slow_wiggle_same_final_frame` [ignored]

- **场景**: 相同 resize 序列，快速 vs 慢速应得到相同的最终 framebuffer。
- **方法**: `run_mode` 闭包参数化 fast/slow 路径——slow 路径每步等待帧到达，fast 路径直接发送所有 resize 请求。
- **注意**: 代码编辑器 UI 的不确定性，标记为 `#[ignore]`。

#### `terminal_makepad_tui_fast_wiggle_preserves_framebuffer`

- **场景**: 使用 `makepad-tui-test` 二进制测试 TUI 程序的快速 resize 稳定性。
- **流程**: 启动 TUI → 记录 baseline → 6 轮快速 wiggle（10-30 行）→ 验证最终 framebuffer 与 baseline 一致。
- **注意**: 需要 `target/release/makepad-tui-test` 二进制预编译。

#### `terminal_makepad_tui_fast_resize_during_output_matches_no_resize` [ignored]

- **场景**: 快速 resize + 并发 TUI 输出 vs 慢速 resize 应得到相同最终帧。
- **方法**: 10 轮 resize 爆发 + TUI "hi" 输出 → 快速和慢速路径对比。
- **注意**: 标记为 stress diagnostic。

#### `terminal_makepad_tui_fast_then_slow_same_session_matches_slow_only` [ignored]

- **场景**: 同一 TUI 会话中，快 resize 后继续慢 resize 的结果应和纯慢 resize 一致。
- **验证**: 快速 resize 的短暂损坏不应持久化到后续慢 resize 中。
- **注意**: 标记为 stress diagnostic。

---

### 3. 文件系统与文件监控 (lines 1456-2749)

#### `file_tree_keeps_hidden_directories_for_backend`

- **场景**: 后端文件树应包含 `.hidden` 目录及其文件（后端需要看到所有文件）。
- **验证**: `.hidden`、`.hidden/secret.txt` 和 `src/lib.rs` 均在节点列表中。

#### `unmount_emits_file_tree_diff_scoped_to_mount`

- **场景**: Unmount 操作触发 `FileTreeDiff`，且 diff 仅包含被卸载 mount 的路径变化。
- **验证**: `changes` 中所有路径以 `"alpha"` 开头，没有 `"beta"` 路径。

#### `file_watch_emits_single_path_delta_without_full_tree_reload`

- **场景**: 文件保存后仅发送 `FileTreeDiff`（增量），不触发完整 `FileTree` 重载。
- **验证**: `SaveTextFile` → 收到 `FileTreeDiff`（1 个 change）→ 后续 drain 消息中无 `FileTree` 出现。

#### `save_text_file_does_not_echo_file_changed_to_saving_client`

- **场景**: 保存文件的客户端不应收到自己操作触发的 `FileChanged` 事件。
- **验证**: `SaveTextFile` → 收到 `TextFileSaved`，但无 `FileChanged`。

#### `file_watch_ignores_makepad_term_writes`

- **场景**: `.makepad/` 目录下的 `.term` 文件写入不触发文件树 diff。
- **验证**: 保存 `.makepad/a.term` 后检查无 `FileTreeDiff`。

#### `file_watch_emits_hidden_directory_writes`

- **场景**: `.hidden/` 目录中的写入应正常触发 `FileTreeDiff`。
- **验证**: 保存 `.hidden/a.txt` → 收到 `FileTreeDiff` 包含该路径。

#### `file_watch_picks_up_external_new_file`

- **场景**: 外部创建的新文件应被文件监控检测到。
- **验证**: 通过 `fs::write` 创建 `src/new_file.rs` → 轮询收到 `FileTreeDiff`（Added）或完整 `FileTree`。

#### `file_watch_emits_file_changed_for_external_write`

- **场景**: 外部修改文件（非通过 `SaveTextFile`）应通知 UI 客户端。
- **验证**: 外部 `fs::write` 修改 `src/lib.rs` → 收到 `FileChanged` 事件。

#### `file_watch_picks_up_external_removed_directory`

- **场景**: 外部删除目录后，文件树应反映删除。
- **验证**: 删除 `src/nested/` → 轮询收到 `FileTreeDiff`（Removed）或完整 `FileTree` 中不再包含该路径。

#### `read_text_file_returns_fresh_content_after_external_write`

- **场景**: 外部修改文件后，`ReadTextFile` 应返回最新磁盘内容（不命中缓存）。
- **验证**: `OpenTextFile` 读初始内容 → 外部修改文件 → `ReadTextFile` 读新内容。

---

### 4. RunItem 和 Splash 脚本 (lines 1564-2321)

#### `run_items_are_pushed_per_mount`

- **场景**: 每个 mount 的 `makepad.splash` 独立定义 RunItems，ObserveMount 后应收到各自 mount 的 RunItems。
- **验证**: 两个 mount 分别定义 "alpha-app" 和 "beta-app"，各自收到 `RunItems { mount }` 消息。

#### `run_item_executes_named_on_run_callback`

- **场景**: `RunItem` 消息触发 splash 脚本中对应 item 的 `on_run` 回调。
- **验证**: `RunItem { name: "hello" }` → splash 脚本执行 `std.println("hello from item")` → `QueryLogs` 检索到该日志。

#### `run_item_spawns_cargo_run_for_clicked_name`

- **场景**: `RunItem` 消息触发 cargo 构建流程（通过 `hub.run` 调用 cargo）。
- **验证**: 收到 `BuildStarted`（包含 package 名）→ 收到 `BuildStopped`（exit_code 0）→ `QueryLogs` 检索到 "hello from clicked item" 输出。

#### `run_item_binds_self_to_registered_item`

- **场景**: `self` 变量在 `on_run` 回调中绑定到注册的 runnable item。
- **验证**: `hub.set_run_items` 中设置 `"package":"self-bound"`，`on_run` 中 `self.package` 打印 "self-bound"。同时验证 `me` 为 nil（不是从 UI 点击触发的）。

#### `run_item_reports_script_error_in_hub_run_args`

- **场景**: `hub.run` 调用参数错误（如传入 nil env）时不应启动构建进程。
- **验证**: `RunItem` 执行后 500ms 内无 `BuildStarted` 消息。

#### `splash_runnable_prints_hello`

- **场景**: `Run` 消息直接执行 `makepad.splash` 脚本。
- **验证**: 脚本执行 `std.println("hello")` → `QueryLogs` 检索到 "hello"。

#### `observe_mount_auto_starts_splash`

- **场景**: `ObserveMount` 应自动启动 mount 目录下的 `makepad.splash`。
- **验证**: `ObserveMount` 后查询日志，应包含 splash 脚本的 "hello" 输出。

#### `observe_mount_reload_splash_after_save`

- **场景**: 保存 splash 文件后，Hub 自动重新加载并执行 splash 脚本。
- **验证**: 原 splash 输出 "one" → `SaveTextFile` 改为 "two" → 日志中出现 "two" 输出。

#### `observe_mount_recovers_splash_after_error_on_followup_save`

- **场景**: Splash 脚本出错后（语法错误），后续保存正确的 splash 应恢复 run items。
- **验证**: 正确 splash（定义 "one"）→ 破坏 splash（语法错误）→ RunItems 变空 → 恢复 splash（定义 "two"）→ RunItems 恢复（包含 "two"）。

#### `external_makepad_splash_fix_restarts_failed_splash`

- **场景**: 外部文件系统修改（`rename` 原子替换）修复 splash 后，Hub 应检测到变化并恢复。
- **验证**: 破坏 splash → RunItems 变空 → 外部原子替换修复 → RunItems 恢复（包含 "two"）。

---

### 5. 搜索与文本操作 (lines 2752-2904)

#### `find_in_files_defaults_to_rs_md_toml_and_returns_concise_hits`

- **场景**: 默认文件搜索限制在 `.rs`、`.md`、`.toml` 文件，返回简洁的行文本。
- **验证**: 搜索 "needle" → 结果中应有 `src/lib.rs` 和 `README.md`，不应有 `notes.txt`，所有 `line_text` 非空。

#### `find_in_files_regex_respects_max_results`

- **场景**: 正则搜索应尊重 `max_results` 上限。
- **验证**: 搜索 "needle"（max_results=1）→ 结果仅 1 条。

#### `read_text_range_returns_line_window_and_total_line_count`

- **场景**: `ReadTextRange` 返回指定行范围的内容和文件总行数。
- **验证**: 4 行文件 → 请求 2-3 行 → 返回 "line-2\nline-3"，`total_lines` 为 4。

---

## 测试模式总结

| 模式 | 说明 |
|------|------|
| **In-process 模式** | 所有测试均使用 `StudioHub::start_in_process`，在同一进程内通过 `HubConnection` 通信，无需网络。 |
| **轮询等待** | 基于 deadline 循环 + `recv_timeout` 的消息等待模式，支持超时。 |
| **终端 framebuffer 解码** | `framebuffer_to_text` 函数将 PTY 的 raw cell 数组解码为可读文本，用于断言终端内容。 |
| **参数化闭包** | `run_mode` 闭包用于在同一测试中对比 fast/slow 路径的结果。 |
| **忽略标记** | 依赖外部二进制（`codex`、`makepad-tui-test`）或非确定性 UI 行为的测试标记为 `#[ignore]`。 |
| **条件编译** | 终端测试仅限 `cfg(unix)`。 |
| **文件监控** | 使用 `notify` 库的底层文件事件监听，测试外部写入、删除、重命名。 |
