# ai_manager_test.rs

## 概述

测试 AI Manager 模块的完整功能：AI 聊天后端通信、流式 SSE 解析、工具调用（read_file、open_editor、observe_filesystem）、聊天持久化、重连恢复、Thinking 内容流式传输和连续多轮对话。

## 测试基础设施

- `ai_env_lock()`: 全局 Mutex 锁，确保 AI 测试串行执行（避免 `MAKEPAD_STUDIO_AI_BASE_URL` 环境变量冲突）。
- `EnvGuard`: RAII 守卫，设置环境变量并在 Drop 时恢复原值。
- `read_http_request(stream)`: 手动解析 HTTP 请求（含 Content-Length），读取完整请求体。
- `write_chunked_sse(stream, chunks)`: 模拟 OpenAI 兼容的 SSE（Server-Sent Events）流式响应，使用 chunked transfer encoding。
- `write_chunked_sse_and_hold_open`: 发送 SSE 后保持连接打开一段时间（用于测试连接复用）。
- `wait_for_ai_state(connection, mount, timeout, predicate)`: 轮询 `HubToClient::AiMountState` 消息直到满足指定条件。
- `wait_for_message(connection, timeout, predicate)`: 通用消息等待。

### 测试架构

每个 AI 测试启动一个本地 `TcpListener`（绑定到 `127.0.0.1:0`），在一个独立线程中扮演 OpenAI 兼容的 API 服务器。Hub 后端配置 `MAKEPAD_STUDIO_AI_BASE_URL` 指向该本地服务器。

```text
[Test Thread] --TcpListener--> [AI Server Thread] --SSE chunks--> [Test Thread]
     |                              ^
     |  ClientToHub::AiSendPrompt   |
     v                              |
[StudioHub] --HTTP POST SSE--> [Local AI Server]
     |
     |  HubToClient::AiMountState (via HubConnection)
     v
[Test Thread asserts state]
```

---

## 测试场景

### `ai_manager_round_trips_prompt_through_local_backend`

- **场景**: 完整测试 AI 提示的发送/响应流程——发送 prompt → AI 返回 SSE 流 → 响应出现在聊天消息中。
- **流程**:
  1. 启动本地 AI 服务器（验证 `POST /v1/chat/completions` 和 `"stream":true`）。
  2. 发送 `AiGetState` → 确认 `active_agent` 存在，获取 `agent_id`。
  3. 发送 `AiSetBackend { backend_id: "openai_localhost" }` → 确认 state 更新。
  4. 发送 `AiSendPrompt { text: "hello from test" }` → 等待 `pending: true` → 等待 `pending: false` 且消息中包含 "assistant reply"。
  5. 断言消息中有 "hello from test"（用户消息）和 "assistant reply"（AI 回复）。

### `ai_manager_persists_chats_per_mount_and_loads_them_on_restart`

- **场景**: 聊天记录持久化到磁盘（`.makepad/ai_chats/`），重启后端后自动加载。
- **流程**:
  1. 发送 prompt → 收到 assistant reply（包含 `reasoning_content` "thinking now"）。
  2. 创建第二个聊天（`AiCreateAgent`）→ agents 数变为 2。
  3. 关闭连接。
  4. 验证 `ai_chats` 目录下有 2 个 JSON 文件。
  5. 重启后端 → `AiGetState` → 验证 `agents.len() == 2`、标题恢复、`active_agent` 正确。
  6. 切换到第一个聊天（`AiSelectAgent`）→ 验证历史消息完整恢复（包含用户消息和 assistant 回复）。
- **验证重点**: 跨重启的持久化、多聊天切换、`reasoning_content` 存储。

### `ai_manager_executes_tool_calls_inside_the_hub`

- **场景**: AI 返回 `tool_calls`（如 `read_file`），Hub 内部执行工具并将结果注入下一轮请求。
- **流程**:
  1. AI 服务器第一次请求验证包含 `"tools"` 和 `"read_file"`。
  2. AI 返回 `tool_calls`（`read_file` 参数：`Cargo.toml`，offset=1，limit=2）。
  3. AI 服务器第二次请求验证包含 `"role":"tool"`、`"tool_call_id":"call_1"` 和文件内容 "Cargo.toml"。
  4. 最终消息中包含 "read_file" 和 "finished after tool call"。
- **验证重点**: Hub 自动执行 `read_file` 工具，读取文件内容并在下一轮 API 调用中作为 tool result 发送。

### `ai_manager_open_editor_tool_opens_file_in_primary_ui`

- **场景**: `open_editor` 工具调用使 Hub 向 primary UI 发送 `TextFileOpened` 消息。
- **流程**:
  1. 设置 `ObserveMount { primary: Some(true) }`。
  2. AI 返回 `tool_calls`（`open_editor`，路径 `src/lib.rs`）。
  3. Hub 向 primary UI 发送 `TextFileOpened`（包含文件内容和路径）。
  4. AI 第二次请求中包含 "Opened repo/src/lib.rs in Studio editor."。
  5. 断言最终聊天消息中包含 "open_editor"。
- **验证重点**: `open_editor` 工具会同时打开编辑器并通知 AI 工具执行成功。

### `ai_manager_open_editor_tool_forwards_jump_location_to_primary_ui`

- **场景**: `open_editor` 工具携带 `line` 和 `column` 参数时，这些位置信息应转发给 UI。
- **流程**:
  1. AI 返回 `tool_calls`（`open_editor`，参数含 `line:1, column:8`）。
  2. 验证 `TextFileOpened` 消息包含 `line: Some(1)` 和 `column: Some(8)`。
  3. AI 第二次请求中包含 "Opened repo/src/lib.rs at 1:8 in Studio editor."。
- **验证重点**: 编辑器跳转位置的精确传递。

### `ai_manager_observe_filesystem_tool_reports_recent_changes`

- **场景**: `observe_filesystem` 工具报告指定路径下的近期文件系统变更。
- **流程**:
  1. AI 返回 `tool_calls`（`observe_filesystem`，参数 `src`，`since_secs: 300`）。
  2. 测试先通过 `fs::write` 修改文件，等待 `FileChanged` 事件。
  3. AI 第二次请求验证包含 `src/lib.rs` 变更信息。
- **验证重点**: Hub 的文件监控事件累积并可通过 `observe_filesystem` 工具提供给 AI。

### `ai_manager_streams_thinking_before_completion`

- **场景**: AI 返回的 `reasoning_content`（thinking）在最终回复前就被流式推送到 UI。
- **流程**:
  1. AI SSE 先发 `reasoning_content: "thinking now"`，再发 `content: "hello"`。
  2. 验证 `pending: true` 状态下消息中包含 `AiMessageRole::Thinking`（"thinking now"）。
  3. 验证最终消息中包含 `AiMessageRole::Assistant`（"hello"）。
- **验证重点**: Thinking 内容在 assistant 回复前实时流式推送。

### `ai_manager_preserves_streamed_thinking_whitespace`

- **场景**: 分块流式的 `reasoning_content` 在拼接时应保留空格（不丢失单词间空白）。
- **流程**:
  1. AI SSE 分三次发送 `reasoning_content`："The"、" user"、" says hi"。
  2. 验证最终 thinking 文本为 "The user says hi"（空格保留）。
- **验证重点**: 流式 thinking 片段的拼接正确性。

### `ai_manager_accepts_second_prompt_after_done_before_socket_close`

- **场景**: 第一个 AI 回复完成后（`[DONE]`），在连接关闭前可发送第二个 prompt。
- **流程**:
  1. 第一个 prompt → AI 返回 "Hi!"（连接保持 2 秒）。
  2. 第二个 prompt → AI 返回 "Roses are red"。
  3. 验证最终消息中包含两个 assistant 回复。
- **验证重点**: 同一个 AI 连接的复用和连续对话支持。

---

## 测试模式总结

| 模式 | 说明 |
|------|------|
| **本地 AI 服务器模拟** | 每个测试启动一个 `TcpListener` + `thread::spawn` 模拟 OpenAI API，精确控制 SSE 流内容和时序。 |
| **全局互斥锁** | `ai_env_lock()` 确保 AI 测试串行化，避免环境变量竞争。 |
| **RAW HTTP 解析** | `read_http_request` 手动解析 HTTP 请求（包括 Content-Length），不依赖 HTTP 库。 |
| **Chunked SSE** | `write_chunked_sse` 使用 `Transfer-Encoding: chunked` 模拟真实的 SSE 流式传输。 |
| **状态轮询** | `wait_for_ai_state` 通过轮询 `AiMountState` 消息跟踪 AI 状态变化（pending、messages、agents 列表）。 |
| **双工通信** | 测试线程 ↔ Hub ↔ 本地 AI 服务器，三方交互验证。 |
| **持久化验证** | 验证 `.makepad/ai_chats/*.json` 文件的存在和数量。 |
| **工具调用编排** | 验证 Hub 自动执行 tool_calls → 收集结果 → 注入下一轮请求的完整循环。 |
