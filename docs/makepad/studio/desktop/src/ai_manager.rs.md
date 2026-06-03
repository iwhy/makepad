# `ai_manager.rs` — AI Manager 面板管理

## 模块作用

管理 Studio Desktop 中的 AI 交互面板（AiPane），包括 agent 的创建/删除/选择、prompt 的发送与取消、聊天 Markdown 渲染、活动状态显示等。

## 常量

- `AI_CHAT_SCROLL_SETTLE_FRAMES = 4` — 滚动到底部所需帧数
- `AI_TASK_EVENT_PREFIX = "TASK EVENT:"` — 任务事件前缀
- `AI_WAITING_MESSAGE_PREFIX = "WAITING:"` — 等待消息前缀
- `AI_TERMINAL_OBSERVATION_PREFIX = "TERMINAL OBSERVATION:"` — 终端观察前缀
- `AI_CHAT_COMPACT_MAX_CHARS = 220` — 紧凑模式最大字符数
- `AI_CHAT_ACTIVITY_MAX_CHARS = 140` — 活动条目最大字符数

## 核心方法

### Agent 管理

**`init_ai_manager`**: 启动时对每个 mount 发送 `AiGetState` 请求，并同步 widget 显示。

**`receive_ai_state`**: 存储 `AiMountState`，如果 mount 有消息则自动滚动到底部。

**`create_ai_manager_agent` / `delete_ai_manager_agent`**: 发送 `AiCreateAgent` / `AiDeleteAgent`。

**`select_ai_manager_agent`**: 从 dropdown 中选择 agent，发送 `AiSelectAgent`。

### Prompt 发送/取消

**`send_ai_manager_prompt`**: 读取输入框文本，检查 agent 是否 pending，调用 `send_ai_prompt_to_agent`。

**`cancel_ai_manager_prompt`**: 发送 `AiCancelPrompt`。

**`active_ai_agent_is_pending` / `active_ai_agent_is_pending_for_mount`**: 检查当前 agent 是否正在处理中。

### 同步 UI

**`sync_ai_manager_widgets`**: 完整刷新 AI 面板：
1. 设置 live_markdown（实时状态显示）
2. 如果没有 AI 状态，显示"Loading AI..."并将 run 按钮设为禁用
3. 否则：
   - 更新 agent dropdown 标签（pending agent 标题后加 `*`）
   - 设置聊天 Markdown 内容
   - 更新状态标签
   - 设置 run 按钮文本（pending → `■`，ready → `▶`）和启用状态

### 滚动管理

**`schedule_ai_chat_scroll_to_bottom`**: 启动多帧滚动动画（4 帧）。

**`flush_ai_chat_scroll_to_bottom`**: 每帧递减计数器，直到 0 时停止滚动。

**`scroll_ai_chat_to_bottom`**: 将 `chat_scroll` 的 scroll position 设为 `(0, 1000000)`。

### Prompt 发送

**`send_ai_prompt_to_agent`**: 核心发送逻辑：
1. 检查 agent 是否 pending → 拒绝发送
2. 如果 `echo_local`，在本地先追加用户消息和空白 thinking 消息（实现即时反馈）
3. 更新 agent 标题（如果是首次消息且标题以 "Chat " 开头，使用 prompt 摘要作为新标题）
4. 发送 `AiSendPrompt`

## Markdown 渲染

### `ai_chat_markdown`
将 agent 的消息列表渲染为 Markdown 字符串：
- 主消息（User/Assistant/System/Thinking/Tool/Error）→ `### {Role}` 标题 + 正文
- 活动消息（Thinking/Waiting/Observation/Tool/Event）→ 紧凑活动区块

### `ai_main_message_heading`
根据角色返回 Markdown 标题：
- `User` → `### User`
- `Assistant` → `### Assistant`
- `System` → `### System`
- `Thinking` → `### Thinking`
- `ToolCall | ToolResult` → `### Tool`
- `Error` → `### Error`

### `ai_activity_item`
判断消息是否需要作为活动条目渲染：
- **Thinking 角色** → 如果以 `WAITING:` 开头则为 Waiting，否则为 Thinking
- **ToolCall/ToolResult** → 工具调用活动
- **User 角色 + TASK EVENT 前缀** → 事件
- **System 角色 + TERMINAL OBSERVATION 前缀** → 终端观察

### `append_activity_markdown`
渲染活动条目区块：
- 所有 Tool 活动合并为内联显示（`> **Tools** - text - text x2`）
- Thinking/Waiting/Observation/Event 使用代码块显示（`> **{Kind}**\n\n\`\`\`runsplash\n...\n\`\`\``）

### `compact_activity_runs`
合并连续相同类型和文本的活动条目（例如连续多次的 `read_terminal` 调用合并为 "Read terminal x2"）。

### `summarize_tool_call_message` / `summarize_tool_result_message`
工具调用的智能摘要：
- `read_terminal`：调用结果返回空字符串（不显示），失败则显示 "Read terminal failed"
- `send_terminal_text`：区分是否按了 Enter
- `send_terminal_key`：显示路径
- `observe_filesystem`：显示检查了多少个更改
- `open_editor`：直接显示 payload 文本

### `summarized_chat_title`
如果 chat 标题以 "Chat " 开头且没有消息，使用第一个 prompt 的前 40 个字符作为新标题。

## 活动条目数据结构

### `AiActivityKind`
```rust
enum AiActivityKind {
    Thinking,     // AI 正在思考
    Waiting,      // 等待（终端输出等）
    Observation,  // 终端观察
    Tool,         // 工具调用
    Event,        // 任务事件
}
```

### `AiActivityItem` / `AiActivityRun`
- `AiActivityItem` — 单个活动条目（kind + text）
- `AiActivityRun` — 条目在 compact 后可能包含 count（重复次数）

## 本地 Prompt Echo

### `apply_local_prompt_echo`
在发送 prompt 前在本地状态中立即追加用户消息和空白 thinking 消息，提供即时 UI 反馈。同时更新 agent summary 中的 title、pending 和 message_count。

## 工具函数

- `clean_activity_text`：按空格分割再合并（规范化空白）
- `normalize_activity_block_text`：去除空行并 trim 每行
- `sanitize_fenced_text`：将 ` ``` ` 替换为 ` ''' `（防止嵌套代码块）
- `truncate_inline`：截断文本到指定字符数并追加 `…`
- `non_empty_labels`：确保 label 列表不为空

## 单元测试

- `apply_local_prompt_echo_updates_visible_agent_immediately` — 验证 prompt echo 正确更新 agent 状态
- `ai_chat_markdown_renders_waiting_messages_as_waiting` — 等待消息渲染
- `ai_chat_markdown_renders_terminal_observation_messages` — 终端观察渲染
- `read_terminal_tool_messages_are_compact` — 工具消息紧凑显示
- `ai_chat_markdown_groups_activity_before_assistant` — 活动条目在 Assistant 消息前分组
- `ai_chat_markdown_does_not_hide_activity_behind_more_count` — 不隐藏重复活动
