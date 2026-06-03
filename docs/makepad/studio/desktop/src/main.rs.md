# `main.rs` — Makepad Studio Desktop 主入口

## 模块概览

`main.rs` 是 Makepad Studio Desktop 应用的入口点和核心调度器。它聚合了所有子模块，定义了顶层 `App` 结构体，并通过 `app_main!` 宏接入 Makepad 框架。

### 模块声明与重导出

- 公开子模块：`ai_manager`, `app_data`, `app_ui`, `desktop_code_editor`, `desktop_file_tree`, `desktop_log_view`, `desktop_profiler_view`, `desktop_run_list`, `desktop_run_view`, `desktop_terminal_view`
- 内联子模块（`#[path]`）：`app_backend`, `app_messages`, `app_state`, `app_tabs` — 它们虽作为独立文件存在，但在模块树上作为 `crate::` 的直接子模块注册
- 重导出依赖 crate：`makepad_code_editor`, `makepad_studio_hub`, `makepad_widgets` 及其子 crate
- 引入 `makepad_studio_protocol::hub_protocol` 中的消息类型（`ClientToHub`, `HubToClient`, `LogEntry`, `QueryId` 等）

## `register_script_modules` 函数

以固定顺序依次注册所有 widget 模块的 `script_mod`。顺序至关重要——必须在使用前完成注册。

## `App` 结构体

```rust
#[derive(Script, ScriptHook)]
pub struct App {
    #[live] ui: WidgetRef,           // 根 UI 引用
    #[rust] data: AppData,           // 全局应用状态
    #[rust] file_filter_debounce_timer: Timer,  // 文件过滤防抖定时器
    #[rust] pending_file_filter: Option<(String, String)>,  // 待发送的过滤请求
    #[rust] sidebar_animation: Option<SidebarAnimation>,    // 侧边栏动画状态
    #[rust] sidebar_animation_next_frame: NextFrame,
    #[rust] bottom_panel_animation: Option<BottomPanelAnimation>,  // 底部面板动画
    #[rust] bottom_panel_animation_next_frame: NextFrame,
    #[rust] ai_chat_scroll_pending: bool,       // AI 聊天滚动待处理
    #[rust] ai_chat_scroll_next_frame: NextFrame,
    #[rust] ai_chat_scroll_frames_remaining: u8, // 剩余滚动帧数
}
```

### 生命周期钩子 (`MatchEvent`)

- `handle_startup`: 初始化状态栏标签、启动后端、加载持久化状态、同步 run preview splitter、初始化 AI manager
- `handle_actions`: 处理所有 UI 动作派发

### 事件处理 (`handle_actions` 详细分析)

**侧边栏/面板切换**（第 163-177 行）：
- `sidebar_toggle` 按钮点击 → `toggle_mount_sidebar` 带动画展开/收起侧边栏
- `bottom_panel_toggle` 按钮点击 → `toggle_bottom_panel` 带动画展开/收起底部面板

**文件树操作**（第 179-198 行）：
- `file_clicked` → `open_node_in_editor` 打开文件编辑器
- `filtered_path_clicked` → `open_path_in_editor` 通过过滤结果打开文件
- `file_tree_filter` 文本变化 → `queue_mount_file_filter` 防抖后发送过滤查询

**AI 操作**（第 199-233 行）：
- `ai_agent_dropdown` 选择 → `select_ai_manager_agent`
- `ai_new_button` → `create_ai_manager_agent`
- `ai_delete_button` → `delete_ai_manager_agent`
- `ai_run_button` → 发送/取消 AI prompt
- `ai_prompt_input` escaped → 取消，returned → 发送

**Run 操作**（第 234-242 行）：
- `run_list` 中的 run 请求 → `run_item` 通过 `ClientToHub::RunItem` 启动
- `run_stop_all` → `request_stop_all_builds_for_mount`

**日志操作**（第 243-278 行）：
- `log_tail_toggle` → 切换自动滚动
- 用户手动滚动时自动关闭 tail 模式
- `log_filter` → 防抖过滤
- `clear_log_filter`, `clear_log`, `log_open_profiler` 按钮

**Dock 标签事件**（第 301-379 行）：
- `TabWasPressed` → 切换 mount/editor/run/log/terminal/profiler/AI 标签
- `TabCloseWasPressed` → 根据标签类型分别调用 `close_editor_tab`, `close_run_tab`, `close_log_tab`, `close_profiler_tab`, `delete_terminal_tab_file`
- `ShouldTabStartDrag` / `Drag` / `Drop` → 实现 dock 标签拖拽重组

**编辑器事件**（第 381-389 行）：
- `CodeEditorAction::TextDidChange` → `save_tab_file` 自动保存

### 主事件循环 (`AppMain::handle_event`)

事件处理顺序：
1. 定时器事件 → flushes 文件过滤
2. Escape 键 → 取消 AI prompt
3. `match_event` → 触发 `handle_startup`/`handle_actions`
4. UI handle_event → 携带 `&mut self.data` 作为 scope data
5. NextFrame → 驱动侧边栏/面板动画和 AI 聊天滚动
6. `WindowDragQuery` → 防止标题栏按钮被窗口拖动捕获
7. Signal → `drain_studio_messages` 从 hub 拉取消息
8. `refresh_run_view_targets` → 更新所有 RunView 的帧请求目标
9. `save_state_if_needed` → 检查 dock 状态变化并持久化

## 辅助函数

### `push_capped_deque`
向 `VecDeque` 追加元素，超过 `max_len` 时从头部弹出，用于日志条目限流。

### `SidebarAnimation` / `BottomPanelAnimation`
面板切换动画描述，包含 mount 名称、起止尺寸和时间戳。

### `parse_path_line_column_token`
从空格分隔的 token 中解析 `path:line:column` 三元组，用于从日志消息中提取文件位置。

### `path_to_virtual`
将 `std::path::Path` 转换为虚拟路径（去除根目录组件，用 `/` 连接）。

## UI 结构

顶层 script_mod 定义了：
```
Root
 └── main_window := AppUI {}   // app_ui.rs 定义
```

## 与 Hub 的交互

App 通过 `send_studio(ClientToHub::...)` 发送消息到后台 hub，通过 `Event::Signal` 接收 `HubToClient` 消息队列。这是典型的 actor 模式——UI 线程与后台服务通过通道通信。
