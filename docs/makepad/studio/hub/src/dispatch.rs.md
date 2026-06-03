# `dispatch.rs` — 核心事件调度器 (6524 行)

## 文件位置
- 路径: `studio/hub/src/dispatch.rs`
- 行数: 6524 行
- 作用: Hub 的核心事件循环，处理所有 Client/App/BuildBox 消息、构建管理、脚本执行、AI 交互、文件变更通知

---

# 第一部分：模块结构和核心数据 (~1-1200 行)

## 引入依赖

- `crate::build_manager::BuildManager`
- `crate::script_manager::{ScriptManager, ScriptId, MAKEPAD_SPLASH_RUNNABLE}`
- `crate::ai_manager::AiManager`
- `crate::terminal_manager::TerminalManager`
- `crate::log_store::{LogStore, ProfilerStore}`
- `crate::virtual_fs::VirtualFs`
- `crate::worker_pool::WorkerPool`
- `makepad_studio_protocol::hub_protocol::*`（所有 hub 协议类型）

## `HubEvent` — 核心事件枚举

```rust
pub enum HubEvent {
    // 连接事件
    ClientConnected { web_socket_id, sender, typed_sender },
    ClientDisconnected { web_socket_id },
    AppConnected { build_id, crate_name, web_socket_id, sender },
    AppDisconnected { web_socket_id },
    BuildBoxConnected { web_socket_id, sender },
    BuildBoxDisconnected { web_socket_id },
    
    // 数据事件
    ClientEnvelope { web_socket_id, envelope: ClientToHubEnvelope },
    ClientBinary { web_socket_id, data },
    ClientText { web_socket_id, text },
    AppBinary { web_socket_id, data },
    BuildBoxBinary { web_socket_id, data },
    
    // 构建/进程事件
    ProcessOutput { build_id, is_stderr, line },
    ProcessExited { build_id, exit_code },
    ProcessAppMessage { build_id, msg: AppToStudio },
    
    // 脚本事件
    ScriptOutput { script_id, mount, is_stderr, line },
    ScriptExited { script_id, mount, exit_code },
    ScriptRunRequest { child_build_id, mount, cwd, program, args, env, package },
    RunItemsUpdated { mount, items },
    
    // 终端事件
    TerminalOutput { path, data },
    TerminalExited { path, exit_code },
    TerminalResized { path, cols, rows },
    
    // 构建盒事件
    BuildBoxCommand { build_id, command: BuildBoxCommand },
    BuildBoxOutput { build_id, is_stderr, line },
    BuildBoxExited { build_id, exit_code },
    
    // 文件系统
    FsChange { changes: Vec<FsChange> },
}
```

## `HubCore` — 核心结构体

```rust
pub struct HubCore {
    // 事件通道
    event_rx: Receiver<HubEvent>,
    event_tx: Sender<HubEvent>,
    
    // 客户端管理
    next_client_id: u64,
    clients: HashMap<u64, ClientConnection>,
    client_id_by_ws_id: HashMap<u64, ClientId>,
    
    // 应用连接管理
    next_app_id: u64,
    app_connections: HashMap<u64, AppConnection>,
    
    // BuildBox 连接
    buildbox_connections: HashMap<u64, BuildBoxConnection>,
    
    // 系统服务
    vfs: VirtualFs,
    builds: BuildManager,
    scripts: ScriptManager,
    ai: AiManager,
    terminals: TerminalManager,
    logs: LogStore,
    profiler: ProfilerStore,
    search_pool: WorkerPool,
    
    // 构建/任务 ID 映射
    build_tasks: HashMap<(String, String), Vec<QueryId>>,
    workspace_mount_path: Option<PathBuf>,
    
    // 文件系统变更合并
    fs_coalesce_state: FsCoalesceState,
    
    // 广播
    client_sockets: HashMap<u64, Sender<Vec<u8>>>,
    typed_client_senders: HashMap<u64, Sender<HubToClient>>,
    
    // AI connector state
    ai_connector_tx: Option<mpsc::Sender<AiConnectorToHub>>,
    ai_connector_jh: Option<JoinHandle<()>>,
    
    // Headless 模式下无需等待 AI connector
    no_ai_connector: bool,
}
```

### `HubCore::new()`
构造函数，初始化所有管理器、线程池、VF S等，计算 workspace mount path。

### `HubCore::run()` — 主事件循环
```rust
pub fn run(&mut self) {
    for event in self.event_rx {
        self.handle_event(event);
    }
}
```
同步阻塞循环，逐条处理 `Receiver<HubEvent>` 中的事件。

---

# 第二部分：事件处理核心 (~1200-2000 行)

## `HubCore::handle_event(&mut self, event: HubEvent)`

使用 `match event` 分发到各专用方法。关键分支：

| 事件 | 处理方法 |
|------|---------|
| `ClientConnected` | 分配 `ClientId`，发送 `Hello`，设置 `typed_sender`（进程内 UI） |
| `ClientDisconnected` | 清理客户端（取消所有它的查询、清理 AI conversations） |
| `ClientEnvelope` | 解码 `ClientToHubEnvelope` → `handle_client_message` |
| `AppConnected` | 分配 `AppConnection`，记录 build_id 和 ws_id 映射 |
| `AppDisconnected` | 清理 app 连接 |
| `ProcessOutput` | `Self::handle_process_output` |
| `ProcessExited` | `Self::try_start_script` + 通知 AI |
| `ProcessAppMessage` | 透传给 AI（`Self::handle_app_message_for_ai`） |
| `ScriptOutput` | `Self::handle_script_output` |
| `ScriptExited` | `Self::handle_script_exited` |
| `ScriptRunRequest` | `Self::start_command_run`（从 splash 触发的子进程） |
| `TerminalOutput` | `Self::handle_terminal_output_for_ai` |
| `TerminalExited` | 通知 AI |
| `FsChange` | `Self::coalesce_fs_changes` 合并后广播 |

---

# 第三部分：客户端消息处理 (~2000-3000 行)

## `HubCore::handle_client_message`

解码 `ClientToHub` 分发的巨型 match：

### 查询类
| 消息 | 处理 |
|------|------|
| `FindInFiles` | `execute_find_in_files` → 结果发回客户端 |
| `SearchFiles` | `execute_search_files` → 文件名搜索 |
| `ReadTextFile` | `vfs.read_text_file` |
| `ReadTextRange` | `vfs.read_text_range` |
| `ListBuilds` | `builds.list_builds` |
| `ListMounts` | `vfs.mounts` |
| `QueryLog` | `logs.query(&query)` |
| `QueryProfiler` | `profiler.query(&query)` |
| `WidgetTreeDump` | 转发给 app（进程内 UI → app） |
| `Screenshot` | 转发给 app |
| `ForwardToApp` | 转发给 app |
| `RunItems` | `scripts.invoke_script_run_item` 或直接构建命令 |
| `AiQuery` | `ai.query()` 触发 AI 思考 |

### 操作类
| 消息 | 处理 |
|------|------|
| `SaveTextFile` | `vfs.save_text_file` + FS 变更处理 |
| `TerminalWrite` | `terminals.send_input` |
| `BuildStart` | 启动构建（取决于是否使用 splash） |
| `BuildStop` | `builds.stop_build` |
| `BuildClear` | 停止 + 标记清理（`build_cleared`） |
| `BuildStdin` | `builds.send_stdin` |

### AI 类
| `AiChat` | `ai.chat()` — 对话 |
| `AiNewSession` | `ai.new_session()` |
| `AiCancel` | `ai.cancel_query()` |

---

# 第四部分：构建、脚本、AI 集成 (~3000-4400 行)

## 构建启动流程

`execute_run_item(name, build_id, package_name)`:
1. 如果 splash 脚本运行中 → `scripts.invoke_script_run_item`
2. 否则计算 cargo args（`--package`、`--release`、`--message-format=json`、`--stdin-loop`）
3. 调用 `builds.start_cargo_run` 或 `start_command_run`

## 构建输出处理

`handle_process_output`:
1. 发送 `BuildOutput` 给相关客户端
2. 追加到 `LogStore`
3. 如果 AI 关注此 build → `ai.handle_build_output`
4. 对于 stdout JSON 消息（`AppToStudio`）→ `handle_app_message_for_ai`

## 脚本集成

`try_start_script(build_id, mount_path, cwd, studio_addrs)`:
- 在构建退出后自动尝试为挂载点启动 splash 脚本
- 检查 `makepad.splash` 文件是否存在
- 设置 `MAKEPAD`、`STUDIO_HOST` 等环境变量

## AI 集成

`handle_terminal_output_for_ai`, `handle_app_message_for_ai`:
- 将终端输出和应用消息路由到 `AiManager`
- AI 关注特定 build 时才会接收其输出

---

# 第五部分：文件树变更合并 (~4400-5400 行)

## `FsCoalesceState`

```rust
struct FsCoalesceState {
    pending: HashMap<String, FsChange>,
    timer: Option<std::time::Instant>,
    spawned: bool,
}
```

## 合并算法

`coalesce_fs_changes(changes)`:
1. 遍历所有变更
2. 按路径去重：同一路径的多个变更只保留最后一个
3. 收集所有受影响的 mount
4. 通知文件监视器（`FsWatch`）重启扫描
5. 通知 AI 文件变更
6. 对受影响的 mount 调用 `load_file_tree` 并广播 `FileTreeData` 给客户端
7. 对构建任务自动重启

## 文件树变更触发

`queue_fs_change(path, change_type)`:
- 文件保存（`SaveTextFile`）
- 外部 FS watcher 通知
- AI 工具（写文件、删除文件）

---

# 第六部分：终端输入和 AI 控制台 (~5400-5700 行)

## 控制台输入处理

`handle_console_input_*` 系列方法：
- 处理来自 WebSocket UI 的 `ConsoleInput` 和 `ConsoleInputVec`
- 负责将用户输入路由到 AI 控制台
- 管理终端 AI 交互模式（用户发送消息 → AI 处理 → 返回结果）

---

# 第七部分：FS Watcher (~5700-5800 行)

## 文件系统监视

`start_fs_watch_for_mounts()`:
- 使用 `notify` crate 监视 mount 路径的变更
- 通过 `file_watch_rx` 通道接收通知
- 节流/防抖处理（`FsCoalesceState`）
- 风暴检测：在短时间内大量变更时推迟处理

---

# 第八部分：综合测试 (~5800-6524 行)

## 测试模块结构

测试模块包含大量集成测试，覆盖：

### 客户端连接和 Hello
```rust
#[test]
fn hello_after_connect()
fn client_connects_and_receives_hello_with_id()
fn disconnect_cleans_up_client_state()
```
- 进程内 UI 连接后立即收到 `HubToClient::Hello`
- 断开后清理所有状态

### 文件操作
```rust
fn create_mount_and_read_file()
fn read_text_range_returns_correct_lines()
fn save_text_file_updates_disk()
```
- 挂载创建 → 文件读写 → 范围读取
- 完整 VFS 集成验证

### 搜索功能
```rust
fn find_in_files_literal_search()
fn find_in_files_regex_search()
fn search_files_returns_matching_paths()
```
- Literal（Rabin-Karp）和 Regex 搜索验证

### 构建管理
```rust
fn run_build_then_stop()
fn process_exit_triggers_cleanup()
```
- 构建启动/停止/退出清理

### Terminal 集成
```rust
fn terminal_open_and_resize()
fn terminal_delayed_input()
```
- PTY 终端操作

### AI 交互
```rust
fn ai_query_then_cancel()
fn ai_chat_then_cancel()
fn ai_tool_execution()
fn ai_execute_tool_read_file()
```
- AI 查询/对话/工具执行
- build output 路由到 AI

### 并发和压力
```rust
fn concurrent_builds()
fn multiple_clients_independent_queries()
```
- 多客户端并发测试

### 文件系统变更
```rust
fn fs_change_triggers_file_tree_update()
fn coalesced_fs_changes_batch_update()
```

## 测试辅助模式

```rust
fn with_test_core<F>(test_fn: F) where F: FnOnce(HubConnection, &mut HubCore, Sender<HubEvent>) + Send + 'static
```
- 创建 `HubCore` 实例
- 创建进程内 `HubConnection`
- 等待 `Hello`
- 执行测试函数
- `HubConnection` 的 `send` 方法支持双向通信

## 设计观察

- dispatch.rs 位于事件架构中心，是 hub 中最大最复杂的文件
- 通过 `HubEvent` 枚举解耦所有子模块（build/script/ai/terminal/vfs）
- `HubCore::run()` 是同步循环，所有处理都在一个线程内完成（无需锁）
- AI 集成通过 AiManager 的注入机制（build output → AI context）
- FS watcher 使用风暴检测避免高频文件变更导致的性能问题
- 巨型测试模块覆盖了几乎所有核心路径，不仅是单元测试更是集成测试
