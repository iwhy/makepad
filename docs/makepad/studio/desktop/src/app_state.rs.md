# `app_state.rs` — Studio Desktop 状态持久化

## 模块作用

实现桌面状态的序列化/反序列化，将 dock 布局、编辑器标签、终端标签、侧边栏状态等信息持久化到文件，以便在重启时恢复。

## 持久化数据模型

### `PersistedMountStateRon`
单个 mount 的持久化状态：
- `mount: String`
- `dock_items: HashMap<LiveId, DockItem>` — workspace dock 的布局状态
- `editor_tab_to_path: HashMap<LiveId, String>` — 编辑器标签→路径映射
- `terminal_tab_to_path: HashMap<LiveId, String>` — 终端标签→路径映射
- `sidebar_restore_width: Option<f64>`
- `bottom_panel_restore_height: Option<f64>`
- `file_filter: String`
- `log_filter: String`
- `log_tail: bool`

### `AppStateRon`
顶层持久化状态：
- `active_mount: Option<String>`
- `mount_dock_items: HashMap<LiveId, DockItem>` — 顶层 mount dock 布局
- `mounts: Vec<PersistedMountStateRon>` — 各 mount 的独立状态

## 文件路径

- 新格式：`.makepad/studio_state0.ron`
- 旧格式（回退）：`makepad_state0.ron`

## Dock Items 清理

### `retain_persistable_editor_tabs`
过滤掉终端虚拟路径（`/.makepad/*.term`）的编辑器标签——终端标签不由编辑器持久化机制处理。

### `persistent_workspace_tab_ids`
收集需要持久化的 tab ID 集合，包括固定的预设标签（`tree_tab`, `run_list_tab`, `ai_tab`, `editor_first`, `run_first`, `log_first`, `terminal_first`）和所有编辑器/终端标签。

### `collect_reachable_dock_items`
递归遍历 dock item 树，收集从 `root` 可达的所有 item ID。

### `prune_dock_item`
递归清理 dock items：
- `Tab`：根据 `keep_tab` 闭包决定是否保留
- `Tabs`：递归清理子标签，空容器时删除自身（作为父容器可能存在）
- `Splitter`：递归清理两侧，如果仅剩一侧则用子节点替换自身

### `sanitize_dock_items`
完整清理流程：
1. 从 root 开始 `prune_dock_item` 清理所有不可保留的标签
2. 如果 root 被替换，将新 root 重映射到 `id!(root)`
3. 移除清理后不可达的 items

### `sanitize_mount_dock_items`
仅保留 mount workspace 类型的标签，用于顶层 dock。

### `sanitize_workspace_dock_items`
清理 workspace dock：
1. 保留允许的 tab ID
2. 验证必须的 sidebar tabs（`tree_tab`, `run_list_tab`, `ai_tab`）和 `log_first`/`terminal_first` 存在
3. 如果缺少必要 tab 则返回 None（使用默认布局）

## 持久化恢复

### `rebuild_mount_tab_bindings`
从 dock state 中重建 `tab_to_mount` 映射和各 mount 的 `tab_id`。

### `reset_persisted_ui_state`
清除所有可持久化的 UI 状态（标签、session、日志、profiler 等），为加载新状态做准备。

### `load_state`
完整的状态加载流程：
1. 读取 RON 文件（尝试新格式，回退旧格式）
2. 重置 UI 状态
3. 还原顶层 mount dock 布局
4. 重建 mount tab 绑定
5. 对每个 mount：
   - 还原 workspace dock 布局
   - 还原编辑器/终端标签路径映射
   - 还原工具栏状态（过滤、tail）
   - 还原侧边栏/面板恢复尺寸
6. 更新编辑器标签标题
7. 选中活动 mount
8. 重新打开所有持久化的编辑器文件（发送 `OpenTextFile`）

## 持久化保存

### `save_state`
1. 获取顶层 mount dock 的清理后状态
2. 对每个 mount 收集持久化数据
3. 序列化为 `AppStateRon`
4. 写入 `.makepad/studio_state0.ron`

### `collect_persisted_mount_state`
单个 mount 的状态收集：
- 收集编辑器标签→路径映射（排除终端路径）
- 收集终端标签→路径映射
- 获取 dock 状态并清理（移除关闭的标签）
- 获取侧边栏/面板恢复尺寸和过滤状态

### `save_state_if_needed`
在 `handle_event` 末尾调用。检查顶层 dock 和所有 workspace dock 的 `need_save` 标记，在需要时触发保存。

## 单元测试

### `persisted_mount_state_round_trips_sidebar_restore_width`
验证 `sidebar_restore_width` 序列化/反序列化往返正确。

### `persisted_mount_state_defaults_missing_sidebar_restore_width`
验证旧版持久化数据（缺少该字段）可以被正确反序列化（返回 None）。

### `terminal_editor_tabs_are_not_persisted`
验证终端虚拟路径不会被持久化为编辑器标签。

### `legacy_workspace_dock_without_ai_tab_is_rejected`
验证没有 AI tab 的旧版 dock 状态被拒绝（返回 None）。

### `workspace_dock_with_ai_tab_is_kept`
验证包含 AI tab 的 dock 状态被接受。
