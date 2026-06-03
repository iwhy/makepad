# aichat/src/main.rs

演示 Makepad 的 AI 聊天应用，支持 Streaming 输出、Markdown 渲染、Splash 代码生成和执行、多种 AI 后端切换。

## 整体结构

### UI 定义（第10-261行）

- **第14-175行**：`ChatList` widget 注册 — 包含 PortalList，两种模板：
  - **User** 模板（第27-90行）：蓝色气泡（`#3a5a8a`），包含 selectable Markdown、代码块（CodeView）、Splash 代码渲染、内联/块级数学公式、删除按钮
  - **Assistant** 模板（第92-174行）：灰色气泡（`#2a2a3a`），包含 `RubberView` 流式文本动画（渐变淡入效果）、Markdown + CodeView + Splash + MathView
- **第177-260行**：主 UI — 标题栏含后端选择 DropDown、ChatList、底部的输入框 + Send/Cancel/Clear 按钮、状态标签

### 数据层（第263-339行）

- `ChatData` 全局状态：消息列表、流式文本、流式状态
- `save_to_disk` / `load_from_disk`：JSON 持久化聊天历史

### ChatList widget（第341-410行）

`draw_walk` 核心逻辑：
- 从全局 `CHAT_DATA` 读取消息
- 流式消息使用 `Assistant` 模板，通过 `markdown.set_text()` + `start_streaming_animation()` 实现逐字动画
- 已有消息按角色选择 `User` 或 `Assistant` 模板
- `animating_msg` 跟踪当前动画消息 ID

### AI Backend（第412-466行）

- `claude_splash_system_prompt`：构建包含完整 Splash 文档的系统提示
- `BackendType` 枚举：`ClaudeSplash`（Claude Code 扩展）和 `LocalOpenAi`（本地 OpenAI 兼容 API）
- `BACKENDS` 数组定义可用后端

### 应用逻辑（第468-626行）

- `create_backend_session`：创建 AI Agent 会话
- `clear_chat`：清空消息历史并重建会话
- `send_message`：发送用户消息，创建流式请求，滚动 PortalList 到最新条目
- `cancel_request`：取消正在进行的请求
- `AgentEvent` 处理：`TextDelta`（增量文本 → 更新流式缓冲区 + 重绘）、`TurnComplete`（完成 → 保存消息）、`PromptError`（错误 → 恢复状态）

### 事件处理（第628-758行）

- Send/Cancel/Clear 按钮
- 输入框回车提交、Escape 取消
- 后端切换（重建会话）
- 消息删除（`items_with_actions`）
- `handle_startup`：初始化后端会话
- `after_new_from_script`：从磁盘加载聊天历史
- 主 `handle_event` 循环中处理 AgentEvent

## 关键 API

- `RubberView` — 流式文本动画组件
- `Markdown::set_text()` / `start_streaming_animation()` / `stop_streaming_animation()` / `reset_all_streaming_animations()` / `is_streaming_animation_done()`
- `CodeView` — 代码编辑器（只读模式用于代码块渲染）
- `Splash` — Splash 脚本渲染组件
- `MathView` — LaTeX 数学公式渲染
- `Agent` trait / `AgentEvent` / `SessionConfig` — AI Agent 抽象
- `ClaudeCodeAgent` / `OpenAiBackend` — 具体后端实现
