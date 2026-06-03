# `app_backend.rs` — Studio Desktop 后端管理与辅助方法

## 模块作用

处理后端启动、挂载管理、面板动画、文件过滤、终端管理、mount 切换等基础设施功能。

## CLI 参数解析

### `parse_mounts_spec`
解析 mount 规范字符串。支持两种格式：
- CLI 格式：`name:path,name:path`（逗号分隔，冒号分隔 key-value）
- 环境变量格式：`name=path;name=path`（分号分隔，等号分隔 key-value）

### `parse_cli_arg_value` / `parse_cli_mounts_spec` / `parse_cli_bind_spec`
从命令行参数中提取 `--mounts` 和 `--bind` 参数值。

### `parse_cli_bind_address`
将 `--bind` 参数解析为 `SocketAddr`。支持三种格式：
- 省略 → `127.0.0.1:8001`
- 纯 IP → `{ip}:8001`
- IP:Port → 直接使用

## 面板动画

### `panel_animation_progress`
立方缓动函数，持续 0.16 秒。使用 `1-(1-t)³` 公式实现 ease-out 效果。

### 侧边栏动画（`start_sidebar_animation` / `step_sidebar_animation`）
- 获取当前分屏位置作为起始值
- 创建 `SidebarAnimation` 并请求下一帧
- 每帧计算缓动进度并设置分屏宽度
- 完成后保存状态

### 底部面板动画（`start_bottom_panel_animation` / `step_bottom_panel_animation`）
与侧边栏动画对称，控制垂直方向上的分屏位置。

### `toggle_mount_sidebar`
切换侧边栏展开/折叠。如果当前宽度 ≤ 1px 则展开到保存的宽度（默认 310px），否则折叠并保存当前宽度。

### `toggle_bottom_panel`
切换底部面板展开/折叠。默认恢复高度 220px。

## 文件树管理

### `apply_mount_file_tree_diff`
增量应用文件树变更：
- `Added`：如果节点已存在则更新类型/状态，否则新增
- `Removed`：递归移除节点及其子节点
- `Modified`：更新 git 状态

变更后刷新活动 mount 的文件树和日志面板。

### `sync_mount_tab_bar_visibility`
当 mount 数 ≤ 1 时隐藏顶层 tab bar。

## 后端启动

### `start_backend`
完整的启动流程：
1. 解析当前目录
2. 获取 mount 配置（CLI > 环境变量 > 默认 `makepad:当前目录`）
3. 解析 bind 地址
4. 创建 `HubConfig`（`enable_in_process_gateway: true`）
5. 调用 `StudioHub::start_in_process` 启动进程内 gateway
6. 对每个 mount 发送 `LoadFileTree` 和 `ObserveMount`（`primary: true`）

## UI 更新辅助

- `set_status`：更新状态栏文本
- `set_current_file_label`：更新当前文件路径显示
- `send_studio`：向 hub 发送消息并返回 `QueryId`
- `mount_state` / `mount_state_mut`：按 mount 名称获取/创建状态

## Mount 标签管理

### `ensure_mount_tab`
确保每个 mount 在顶层 dock 中有对应的标签。第一个 mount 使用预定义的 `mount_first`，后续 mount 在其旁边动态创建。

### `mount_from_virtual_path`
从虚拟路径中提取 mount 名称（`/` 分隔的第一个 segment）。

### `terminal_virtual_path` / `is_terminal_virtual_path`
终端文件的虚拟路径格式：`mount/.makepad/xxx.term`。检测条件：路径包含 `/.makepad/` 且以 `.term` 结尾。

### `mount_workspace_widget` / `mount_workspace_dock` / `mount_terminal_dock`
导航工具方法，从 UI 树中获取 mount workspace 的 widget 引用、dock 引用或终端 dock 引用。`mount_terminal_dock` 会在 `terminal_first` 标签不存在时自动创建。

## 面板刷新

### `refresh_active_mount_tree`
重建 `FlatFileTree` 并刷新文件树显示。通过 `take` 临时移出 `FileTreeData` 以避免clone。

### `refresh_active_mount_run_list`
刷新 run list 显示。

### `refresh_active_mount_log_panels`
刷新日志面板和受影响的终端标签。

### `refresh_active_mount_terminal_panel`
刷新单个终端标签。

## 终端管理

### `terminal_tab_title` / `apply_terminal_tab_title` / `reset_terminal_tab_title`
终端标签标题管理（支持后端自定义标题）。

### `terminal_tab_mount_path`
从 tab_id 反向查找 mount 和路径。

## 文件过滤

### `set_mount_file_filter` / `queue_mount_file_filter` / `flush_queued_mount_file_filter`
文件过滤的请求-防抖-执行模式：
1. `queue_mount_file_filter` 取消旧的查询，启动防抖定时器（140ms）
2. `flush_queued_mount_file_filter` 在定时器触发时发送实际查询
3. `set_mount_file_filter` 发送 `ClientToHub::FindFiles` 并记录 query_id

### `cancel_file_filter_query`
取消挂起的过滤请求和旧的查询。

### `redraw_file_tree_if_active`
仅在活动 mount 匹配时刷新文件树。

## 日志管理

### `set_mount_log_tail` / `set_mount_log_filter` / `restart_log_query_for_mount`
日志过滤和自动跟随控制。

### `restart_log_query_for_mount`
取消旧的日志查询，清除所有日志条目，发起新的 `ClientToHub::QueryLogs`。

### `clear_ui_log_entries` / `request_log_clear`
清除 UI 日志条目，或发送清除命令到后端。

## Mount 切换

### `select_mount`
完成整个 mount 切换序列：
1. 设置 `active_mount`
2. 选中顶层 mount tab
3. 刷新文件树（或发送加载请求）
4. 初始化终端文件
5. 应用工具栏状态（过滤、tail 等）
6. 重启日志查询
7. 刷新 run list 和日志面板
8. 请求 AI 状态并刷新 AI 面板

## 终端生命周期

### `ensure_terminal_session_open`
确保终端 session 已打开。如果不在 `terminal_open_paths` 中，发送 `TerminalOpen` 请求。

### `ensure_mount_terminal_file`
扫描文件树中的 `.makepad/*.term` 文件：
1. 收集已有终端文件
2. 移除不再存在的终端 framebuffer（关闭 stale sessions）
3. 同步终端标签到 dock
4. 对每个终端文件确保 session 打开
5. 如果 mount 没有终端文件且未初始化，自动创建默认终端

### `reveal_terminal_path`
展开底部面板并选择到指定终端文件。

### `next_terminal_path`
生成下一个可用终端文件名（`a.term`, `b.term`, ..., `z.term`, `t1.term`, ...）。

### `create_new_terminal_tab`
创建新终端标签的完整流程：
1. 生成新终端路径
2. 存储到 mount state
3. 发送 `SaveTextFile` 创建文件
4. 创建 framebuffer 条目
5. 同步终端标签到 dock
6. 打开 session

### `delete_terminal_path` / `handle_terminal_exit_cleanup` / `delete_terminal_tab_file`
终端删除/退出处理的三种入口，统一调用 `remove_terminal_path_local_state`。

### `remove_terminal_path_local_state`
从所有本地状态中移除终端路径：
1. 关闭编辑器标签（如果存在）
2. 移除终端标签
3. 清理 framebuffer、open_paths、titles 等映射

### `sync_mount_terminal_tabs`
将 mount 的终端文件列表同步到 dock 标签。保持 `terminal_first` 作为锚点，按顺序创建/删除终端标签。

### `select_bottom_terminal_panel` / `select_terminal_path`
底部终端面板的选中逻辑。
