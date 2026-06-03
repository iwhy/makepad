# `app_tabs.rs` — Studio Desktop 标签页管理

## 模块作用

处理所有 dock 标签的创建、切换、关闭、拖拽重组，以及 run/log/profiler/tabs 标签的生命周期管理。所有方法均为 `App` 的 `pub(super)` 方法。

## Splitter 管理

### `run_preview_splitter_is_collapsed` / `run_preview_splitter_restore_target`
判断编辑器/运行预览分屏是否被折叠（`Weighted(≥0.999)`），以及在需要时恢复保存的分屏位置或使用默认值 `0.62`。

### `sync_run_preview_splitter`
同步当前 mount 的 `editor_split` 分屏位置。当存在运行中的 app 时自动展开预览面板，保存当前分屏位置以便后续恢复。

## 标签页标识

### `tab_id_from_widget_uid`
通过 widget 路径获取标签 `LiveId`。路径倒数第二个元素即为 tab_id。

### `set_active_tab`
设置当前活动标签。如果是编辑器标签，更新 `current_file_path` 和文件标签显示。

## 编辑器标签

### `ensure_editor_tab_for_path`
核心方法，确保某个虚拟路径对应的编辑器标签存在：
1. 从路径提取 mount 名称
2. 如果需要，切换到对应 mount
3. 如果标签已存在，直接选中
4. 否则在 `editor_first` 锚点附近创建新标签
5. 更新 `path_to_tab` / `tab_to_path` 映射

### `find_editor_anchor_tab`
查找编辑器标签的锚点位置——优先使用 `editor_first`，否则遍历已有标签。

### `close_editor_tab`
关闭编辑器标签并清理路径映射、session、pending 状态。

### `update_editor_tab_titles`
去重标签标题算法：
1. 对每个标签路径进行分段
2. 从空标题开始，发现重复时逐步增加路径深度（从末尾向前）
3. 直到所有标签标题唯一

### `open_path_in_editor` / `open_node_in_editor`
打开编辑器标签的主入口。如果是终端虚拟路径则转到终端面板，否则发送 `ClientToHub::OpenTextFile` 请求。

### `save_tab_file`
获取当前 session 文本内容，通过 `ClientToHub::SaveTextFile` 保存。

## Dock 工具方法

### `reachable_tab_bar_of_tab`
从 dock 状态树中递归查找某个标签所在的 tab bar 及其位置索引。

### `create_dock_tab`
通用 dock 标签创建方法，在新标签栏中创建/选择标签。

## Run 标签

### `ensure_run_tab_for_build`
为 build 创建或找到对应的 run 标签：
1. 如果已存在且可达，更新 run target 并选中
2. 否则在 `run_first` 锚点附近创建新 `RunningAppPane` 标签
3. 调用 `set_run_target` 设置远程连接目标
4. 调用 `sync_run_preview_splitter` 展开预览面板

### `refresh_run_view_targets`
遍历所有 run 标签，重新设置 `RunView` 的运行目标（build_id, window_id, studio_addr）。

## Log 标签

### `ensure_log_tab_for_build`
类似 run 标签的创建逻辑，在 `log_first` 锚点附近创建 `LogPane` 标签。

## Profiler 标签

### `ensure_profiler_tab_for_build`
在日志面板区域创建 `ProfilerPane` 标签。

### `start_profiler_query_for_build` / `stop_profiler_query_for_build`
启动/停止对指定 build 的持续性能采样查询。使用 `ClientToHub::QueryProfiler`，设置 `live: true` 以接收实时数据。

### `profiler_target_for_mount`
查找 mount 当前的 active log build 或任意日志/运行 build 作为分析目标。

### `open_profiler_for_mount`
一键打开 profiler：获取目标 build，创建标签，启动查询。

## 终端标签

### `send_terminal_input`
确保终端 session 已打开，发送 `ClientToHub::TerminalInput`。

### `request_terminal_viewport`
发送视口请求（cols, rows, pty_rows, top_row）。

## 日志跳转

### `log_jump_position`
将 `(line, column)` 转换为 `CodeSession` 中的 `Position`（行索引 + 字节偏移）。

### `try_apply_log_jump`
尝试在已打开的编辑器标签中跳转到指定位置。

### `apply_pending_log_jump`
文件加载完成后应用之前暂存的跳转请求。

### `open_log_location`
完整流程：打开文件 → 尝试跳转 → 如果文件未加载则暂存跳转位置。

### `extract_log_location` / `virtualize_log_path` / `absolute_to_virtual_path`
从日志条目中解析文件位置。支持三种路径格式：
- 直接虚拟路径（`mount/path`）
- 绝对路径（通过 mount root 解析）
- 相对路径（相对于 mount）

## 清理方法

### `close_mount_run_and_log_tabs`
关闭指定 mount 的所有运行和日志标签。

### `clear_build_tabs`
清除 build 的所有关联标签（run, log, profiler），停止 profiler 查询，清理所有相关状态。

## Action 派发

### `handle_log_view_actions`
处理 `DesktopLogViewAction::OpenLocation` → 调用 `open_log_location`。

### `handle_run_view_actions`
处理 `DesktopRunViewAction::ForwardToApp` → 转发消息到 app。

### `handle_profiler_actions`
处理 `SetRunning`（启动/停止采样）和 `Clear`（清除数据并重启）。

### `handle_terminal_actions`
处理 `Input`（转发字符）和 `RequestViewport`（请求帧数据）。

## 拖拽管理

### `start_workspace_tab_drag`
启动标签拖拽，排除 mount 标签。

### `handle_workspace_tab_drag` / `handle_workspace_tab_drop`
接受/完成标签拖拽，支持跨面板拖拽重组。

## 关闭方法

### `close_run_tab` / `close_log_tab` / `close_profiler_tab`
分别处理各类型标签的关闭，清理状态映射，并发送 `StopBuild` 等通知。

## 单元测试

测试 run preview splitter 的恢复逻辑：
- 无活动运行时不应自动展开
- 折叠状态下有运行时恢复保存的分屏位置
- 无保存状态时使用默认值
