# `ai_manager.rs` — AI Agent 管理器 (4664 行)

## 文件位置
- 路径: `studio/hub/src/ai_manager.rs`
- 行数: 4664 行
- 作用: 实现 AI Agent 的完整状态机和工具执行系统，支持多会话管理、OpenAI API 交互、Agent 子任务委派

---

# 第一部分：核心数据结构 (~1-600 行)

## 全局常量

```rust
const AI_CONNECTOR_MAX_MESSAGE_LENGTH: usize = 5000000;     // 5MB
const AI_CONNECTOR_TIMEOUT_SECS: u64 = 1800;                 // 30 min
const AI_DEFAULT_AGENT_MODEL: &str = "deepseek/deepseek-v4-flash-free";
```
模型默认配置为 `deepseek/deepseek-v4-flash-free`，超时 30 分钟，最大消息 5MB。

## `AiConfig`
```rust
pub struct AiConfig {
    pub agent_model: String,
    pub api_key: String,
    pub api_url: String,
}
```

## `AiSession`
```rust
pub struct AiSession {
    pub session_id: String,
    pub client_id: ClientId,
    pub model: String,
    pub api_key: String,
    pub api_url: String,
    pub messages: Vec<ChatMessage>,
    pub attached_build_ids: HashSet<QueryId>,
    pub pending_tool_calls: Vec<ToolCall>,
    pub pending_tool_results: VecDeque<PendingToolResult>,
    pub max_tool_rounds: usize,
    pub next_sub_agent_id: usize,
    pub sub_agent_parent: Option<(String, ClientId)>,
    pub sub_agents: Vec<String>,
    pub headless: bool,
    pub bypass_confirm: bool,
    pub no_retry: bool,
}
```
支持子 Agent 层级结构（`sub_agent_parent` / `sub_agents`）。

## `ChatMessage`
```rust
pub struct ChatMessage {
    pub role: String,
    pub content: Option<String>,
    pub tool_calls: Option<Vec<ToolCall>>,
    pub tool_call_id: Option<String>,
}
```

## `ToolCall`
```rust
pub struct ToolCall {
    pub id: String,
    pub r#type: String,
    pub function: ToolFunctionCall,
}
pub struct ToolFunctionCall { pub name: String, pub arguments: String }
```

## `PendingToolResult`
```rust
pub struct PendingToolResult {
    pub call: ToolCall,
    pub content: String,
    pub started_at: Instant,
}
```

## `AiTool` — 工具注册表
```rust
pub struct AiTool {
    pub name: &'static str,
    pub description: &'static str,
    pub parameters: serde_json::Value,
    pub handler: fn(
        args: serde_json::Value,
        cx: &mut AiToolContext,
    ) -> Result<serde_json::Value, String>,
}
```
静态注册表，每个工具包含名称、描述、JSON Schema 参数和处理器函数。

## `AiToolContext` — 工具执行上下文
```rust
pub struct AiToolContext<'a> {
    pub vfs: &'a mut VirtualFs,
    pub session: &'a AiSession,
    pub builds: &'a BuildManager,
    pub terminals: &'a mut TerminalManager,
    pub event_tx: &'a Sender<HubEvent>,
    pub logs: &'a LogStore,
    pub config: &'a AiConfig,
    pub workspace_root: Option<&'a PathBuf>,
    pub session_client_id: ClientId,
    pub mount: Option<String>,
    pub mount_root: Option<PathBuf>,
    pub ai_connector_tx: Option<&'a Sender<AiConnectorToHub>>,
}
```
工具可访问的所有资源：VFS、构建管理器、终端、日志、事件通道。

## `AiQuery` / `AiChat` — 协议请求结构
```rust
pub struct AiQuery {
    pub session_id: String,
    pub query_id: QueryId,
    pub model: Option<String>,
    pub prompt: String,
    pub attached_build_ids: Vec<QueryId>,
    pub headless: bool,
    pub bypass_confirm: bool,
    pub api_key_override: Option<String>,
    pub api_url_override: Option<String>,
}
```

---

# 第二部分：AiManager 实现 (~600-1300 行)

## `AiManager`
```rust
pub struct AiManager {
    pub sessions: HashMap<String, AiSession>,
    config: AiConfig,
    worker_pool: WorkerPool,
    vfs: VirtualFs,
    builds: BuildManager,
    terminals: TerminalManager,
    logs: LogStore,
    event_tx: Sender<HubEvent>,
    workspace_root: Option<PathBuf>,
}
```

### 核心方法

| 方法 | 说明 |
|------|------|
| `new(config, worker_count)` | 创建 AiManager，初始化线程池 |
| `query(query, client_id, event_tx)` | 处理 `AiQuery` 请求 → 运行 Agent |
| `chat(chat, client_id, event_tx)` | 处理 `AiChat` 请求 |
| `new_session(session_id, client_id, ...)` | 创建新会话 |
| `cancel_query(query_id)` | 取消运行中的查询 |
| `post_message_to_session(session_id, content, tools, ...)` | 发送消息到 OpenAI API |
| `handle_build_output(session_id, build_id, line, is_stderr)` | 构建输出注入 AI 上下文 |
| `handle_terminal_output(session_id, path, data)` | 终端输出注入 AI 上下文 |
| `handle_app_message(session_id, msg)` | 应用消息注入 AI 上下文 |

### Agent 执行循环

`process_openai_response(session, response, ...)`:
```
发送消息 → 解析响应 → 如果是 tool_calls → 执行工具 → 发送结果 → 重复
```

工具执行循环包含 `max_tool_rounds`（默认 25）限制，防止无限循环。

---

# 第三部分：工具实现 (~1300-2500 行)

## 工具注册

```rust
pub fn register_tools() -> Vec<AiTool> {
    vec![
        read_file_tool(), write_file_tool(), search_files_tool(),
        grep_tool(), move_file_tool(), delete_file_tool(),
        terminal_tool(), run_build_tool(), run_bash_tool(),
        server_fetch_tool(), delegate_tool(),
    ]
}
```

## 工具系统核心

`execute_tool_registry(args, tool_name, ctx)`:
- 在 `args` 中查找 `"tool"` 字段确定工具名
- 调用对应 handler
- 结果序列化为 JSON 字符串

`execute_tool(tool_call, session, ctx, tool_registry)`:
- 解析 `tool_call.function.arguments` JSON
- 调用 `execute_tool_registry`
- 结果包装为 `ChatMessage`（tool role）

### 工具列表

| 工具 | 功能 | 关键参数 |
|------|------|---------|
| `read_file` | 读取文件内容 | `path: string`, `start_line?: int`, `end_line?: int` |
| `write_file` | 写入文件 | `path: string`, `content: string` |
| `search_files` | 按文件名搜索 | `pattern: string`, `mount?: string`, `max_results?: int` |
| `grep` | 内容搜索 | `pattern: string`, `mount?: string`, `is_regex?: bool` |
| `move_file` | 移动/重命名 | `source: string`, `dest: string` |
| `delete_file` | 删除文件 | `path: string` |
| `terminal` | 终端操作 | `command: "open"` / `"write"` / `"close"`, `path`, `data`, `delay_ms` |
| `run_build` | 启动构建 | `mount: string`, `package: string`, `args` |
| `run_bash` | 执行命令 | `command: string` |
| `server_fetch` | HTTP 请求 | `url: string`, `method?: string`, `headers?`, `body?` |
| `delegate` | 委派子任务 | `task: string`, `model?: string`, `api_key?: string` |

#### `terminal` 工具详细

三种子命令：
1. `open` — 打开新终端（指定 `path`, `mount`, `cwd`, `cols`, `rows`）
2. `write` — 写入终端（`path`, `data`, `delay_ms` 用于延迟输入）
3. `close` — 关闭终端

#### `delegate` 工具详细

创建子 Agent 会话：
1. 生成唯一 session_id（`{parent_id}/sub/{seq}`）
2. 创建新的 `AiSession` 复制父会话的 config
3. 在新线程中运行子 Agent
4. 子 Agent 结果通过 `AiConnectorToHub` 通道返回父 Agent

#### `run_bash` 工具

在 Linux 上通过 `sh -c` 执行命令：
- 使用 `Command::new("sh")` + `-c`
- 捕获 stdout/stderr
- 限制输出大小
- 超时 120 秒

#### `server_fetch` 工具

使用 `reqwest` HTTP 客户端：
- 支持 GET/POST/PUT/DELETE/PATCH
- 自定义 headers 和 body
- 超时 60 秒
- 最大响应 10MB
- 阻止访问内网 IP（`blocked_ip` 过滤）

---

# 第四部分：OpenAI API 交互 (~2500-3000 行)

## HTTP 客户端

`post_message_to_session_inner(url, api_key, model, messages, tools, ...)`:
- 构造 OpenAI 兼容的 chat completions 请求体
- 包含 `messages` 数组和 `tools` 定义
- 支持 `bypass_confirm`（确认绕过）、`no_retry`（错误重试控制）
- 使用 `reqwest::Client` 发送 POST 请求
- 反序列化响应为 `OpenAiResponse`

```rust
struct OpenAiResponse {
    choices: Vec<OpenAiChoice>,
    usage: Option<OpenAiUsage>,
}
struct OpenAiChoice {
    message: OpenAiResponseMessage,
    finish_reason: Option<String>,
}
struct OpenAiResponseMessage {
    role: Option<String>,
    content: Option<String>,
    tool_calls: Option<Vec<ToolCall>>,
}
struct OpenAiUsage { total_tokens: Option<u64> }
```

## 特殊标记处理

- `AGENT_USE_OUTPUT` — 构建/终端输出注入标记
- `AGENT_BYPASS_CONFIRM` — 绕过确认
- `<attachment path="...">` — 文件附件注入

---

# 第五部分：AI Connector (~3000-3400 行)

## `AiConnectorToHub` — 子 Agent 到 Hub 通信

```rust
pub enum AiConnectorToHub {
    SubAgentDone {
        parent_session_id: String,
        sub_session_id: String,
        result: String,
    },
    SubAgentError {
        parent_session_id: String,
        sub_session_id: String,
        error: String,
    },
    WriteFileNotification {
        session_id: String,
        path: String,
    },
}
```

## `AiConnector` — 子 Agent 执行器

`run_sub_agent(...)`:
1. 创建新的 `AiSession`（继承父会话配置）
2. 创建 AiManager 副本（共享 VFS 克隆）
3. 在独立线程中执行 Agent 循环
4. 完成或出错后通过通道发送结果

---

# 第六部分：会话管理 (~3400-3800 行)

## 消息上下文构建

`build_context_messages(session, ...)`:
- 将会话中的 `ChatMessage` 列表转换为 API 请求格式
- 注入系统提示（system prompt）：
  - 定义 Agent 角色（编程助手）
  - 工具使用指南
  - 交互规则（确认、读文件、写文件等行为规范）
- 注入文件附件内容（`<attachment>` 标记）
- 注入构建输出（`AGENT_USE_OUTPUT` 标记）

## 工具定义

`build_tools_for_request(session, tool_registry)`:
- 将 `AiTool` 注册表转换为 OpenAI tools API 格式
- 控制最大工具轮次
- 每个工具包含 JSON Schema 参数定义

## 消息注入

`inject_build_output(session, ctx)`:
- 扫描 `session.messages` 中的 `AGENT_USE_OUTPUT` 标记
- 从构建管理器最近的输出收集日志
- 将输出注入为 system 消息

---

# 第七部分：工具实现详解 (~3800-4200 行)

## 各工具 handler 实现

### `read_file_tool`
- 解析 `path` 参数
- 通过 `vfs.read_text_file` 或 `vfs.read_text_range` 读取
- 对非文本文件返回错误

### `write_file_tool`
- 检查是否在 workspace 内
- 通过 `vfs.save_text_file` 保存
- 发送 `WriteFileNotification` 给主 Agent（连接器模式）

### `search_files_tool`
- 调用 `vfs.find_files`
- 返回匹配的文件路径列表

### `grep_tool`
- 调用 `vfs.find_in_files`
- 返回匹配的行结果

### `move_file_tool`
- 读取源文件 → 写入目标文件 → 删除源文件

### `delete_file_tool`
- 检查是否在 workspace 内
- 调用 `vfs.delete_path`

### `terminal_tool`
- `open`: 通过 `ctx.terminals.open_terminal` 打开新 PTY
- `write`: 调用 `ctx.terminals.send_input` 或 `send_input_delayed`
- `close`: 调用 `ctx.terminals.close_terminal`

### `run_build_tool`
- 通过 `ctx.builds.start_command_run` 启动构建
- 返回 build_id

### `run_bash_tool`
- `Command::new("sh")` + `-c` 执行
- 设置 `RUST_BACKTRACE=1`
- 捕获 stdout/stderr（最大 100 行）

### `server_fetch_tool`
- `reqwest::Client` HTTP 请求
- 内网 IP 过滤（私有 IPv4 范围）
- 处理重定向

### `delegate_tool`
- 创建子 Agent 会话
- 将子 Agent 放入 `session.sub_agents`
- 在 `handle_post_message` 中处理子 Agent 结果

---

# 第八部分：会话操作和查询管理 (~4200-4400 行)

## `handle_query` / `handle_chat`

`AiManager::handle_query(query, client_id, event_tx)`:
1. 创建会话（如果不存在）
2. 将用户 prompt 添加到 `session.messages`
3. 调用 `post_message_to_session`
4. 处理响应（工具调用循环）
5. 将最终结果发送到 `HubToClient::AiResult`
6. 发送 token 使用信息

## `AiManager::cancel_query(query_id)`

- 查找正在运行的查询
- 通过 `client_id` 的范围取消（迭代所有会话）
- 标记取消

---

# 第九部分：工具测试 (~4400-4664 行)

## 单工具测试

每个工具都有独立测试函数，使用 `test_support::tempdir()` 创建临时目录：

```rust
fn test_tool_read_file()
fn test_tool_write_file()
fn test_tool_search_files()
fn test_tool_grep()
fn test_tool_move_file()
fn test_tool_delete_file()
fn test_tool_terminal_open_write_close()
fn test_tool_run_build()
fn test_tool_run_bash()
fn test_tool_server_fetch()
fn test_tool_delegate()
```

### 测试模式
1. 创建 `AiManager` 实例（使用测试 config）
2. 设置 work directory 和 VFS mount
3. 创建测试会话
4. 调用工具直接执行函数
5. 断言结果

### `run_bash` 测试
- `echo hello` → stdout 验证
- `cat nonexistent_file` → stderr 验证

### `server_fetch` 测试
- `http://httpbin.org/get` GET 请求验证
- `http://httpbin.org/post` POST 请求验证

### `terminal` 测试
- open terminal → `echo hello` → close → 验证输出

---

# 设计观察

- Agent 使用完全自主循环：发送消息 → 执行工具 → 发送结果 → 继续，直到非 tool_calls 响应
- 最大工具轮次 (`max_tool_rounds` = 25) 防止无限循环
- 子 Agent 委派支持树形任务分解
- 构建/终端输出通过 `AGENT_USE_OUTPUT` 标记注入上下文
- 工具定义使用静态函数指针注册表，非动态分发
- 所有工具可访问完整的 `AiToolContext`，包括 VFS、BuildManager、TerminalManager 等
- 文件操作（write/delete/move）受 workspace 边界检查限制
- 测试覆盖率全面，每个工具都有独立的集成测试
