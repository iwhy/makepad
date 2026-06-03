# `app_messages.rs` — Studio Desktop 消息处理

## 模块作用

处理从 StudioHub 后台接收的所有 `HubToClient` 消息。本文件实现了 `drain_studio_messages` 和 `handle_studio_message` 方法，是桌面前端与后台 hub 通信的核心消息分发器。

## 常量

- `MAKEPAD_SPLASH_RUNNABLE = "makepad.splash"` — 启动画面 runnable 名称，用于过滤不需要显示 UI 的构建。

## 编辑文件同步

### `apply_editor_text_update`
核心编辑内容更新方法：
1. 从 `pending_open_paths` / `pending_reload_paths` 移除待处理标记
2. 如果 `allow_create_tab` 为 false 且路径未打开，跳过更新
3. 通过 `ensure_editor_tab_for_path` 确保标签存在
4. 如果已有 session，仅在内容不同时替换文档内容（避免不必要的重绘）
5. 如果是新 session，创建 `CodeSession` 并插入
6. 应用待处理的日志跳转
7. 触发标签重绘

## 消息循环

### `drain_studio_messages`
循环从 `self.data.studio.try_recv()` 拉取消息，直到队列为空。

## 消息处理详细

### `HubToClient::FileTree`
文件树初始加载完成。将数据存储到 mount state 中，如果存在文件过滤条件则应用过滤。如果是第一个 mount 或当前 active mount，刷新 UI。

### `HubToClient::TextFileOpened`
后端确认文件已打开并返回内容。如果消息附带行列信息则插入 `pending_log_jumps`。

### `HubToClient::FileTreeDiff`
增量文件树更新。使用 `apply_mount_file_tree_diff` 处理增/删/改操作。

### `HubToClient::TextFileRead`
重新读取文件内容（如外部修改）。仅在文件已打开或 pending 时才应用。

### `HubToClient::TextFileSaved`
文件保存结果通知。

### `HubToClient::FileChanged`
外部文件变更通知。处理两种场景：
- **mount 级通知**（路径不含 `/`）：遍历该 mount 下所有打开的文件，逐个发起重新读取
- **文件级通知**：如果不是已打开的文件则忽略，否则加入 pending 队列并发送 `ReadTextFile`

### `HubToClient::FindFileResults`
文件过滤搜索结果。如果结果已过时（query_id 不匹配）则忽略。

### `HubToClient::Builds`
构建列表。如果存在 `pending_stop_all_mount`，则自动停止该 mount 下所有非 splash 的活动构建。

### `HubToClient::RunItems`
runnable 列表更新。刷新 run list 面板。

### `HubToClient::BuildStarted`
构建启动事件：
1. 记录 build→mount 和 build→package 映射
2. 排除 splash runnable
3. 创建 run tab 和 log tab
4. 设置 run target 和 active log build

### `HubToClient::BuildStopped`
构建停止事件：
1. 清除 build→mount 映射
2. 停止 profiler 查询
3. 更新 run tab 中的状态显示
4. 清除 run target

### `HubToClient::BuildCleared`
构建被清除（前端 UI 清理信号）。调用 `clear_build_tabs` 完全移除相关标签。

### `HubToClient::RunViewCreated`
运行窗口创建事件：
1. 查找或创建 run tab
2. 记录 window_id
3. 更新 run target
4. 调用 `rebootstrap_after_app_ready` 触发 swapchain 引导

### `HubToClient::RunViewDrawComplete`
绘制完成通知，包含可呈现的 `PresentableDraw`。更新 run view。

### `HubToClient::RunViewFrame`
远程帧数据（websocket 传输的 PNG/JPEG 帧）。转发到 `DesktopRunView::set_remote_frame`。

### `HubToClient::RunViewCursor`
远程光标更新。

### `HubToClient::RunViewInputViz` / `RunViewKeyFocusRect`
远程输入可视化（点击/键盘焦点矩形显示）。

### `HubToClient::QueryLogResults`
日志查询结果：
1. 验证 `live_log_query` 是否匹配
2. 对每条日志条目应用 `extract_log_location`（从消息中提取文件路径）
3. 将日志追加到 build 日志队列（上限 2000 条）
4. 对 splash runnable 的日志也追到 mount 日志队列（上限 3000 条）
5. 刷新所有受影响的 mount 日志面板

### `HubToClient::QueryProfilerResults`
性能分析采样结果。如果 analysis 已被暂停则忽略。数据驱动 tab 重绘。

### `HubToClient::QueryCancelled`
查询取消通知。清理 `profiler_query_build_by_query` 映射。

### `HubToClient::LogCleared`
日志清除确认。

### `HubToClient::TerminalOpened`
终端打开确认。初始化 framebuffer，创建终端标签，刷新 AI preview。

### `HubToClient::TerminalFramebuffer`
终端帧缓冲更新。按 frame_id 去重。

### `HubToClient::TerminalTitle`
终端标题更新。

### `HubToClient::TerminalExited`
终端进程退出。重置标签标题，清理本地状态。

### `HubToClient::AiMountState`
AI manager 状态更新。直接存储并刷新 AI 面板。

### `HubToClient::Error`
错误消息显示。

## 工具方法

### `parse_run_view_cursor`
将字符串光标名称映射为 `MouseCursor` 枚举。
