# hub_protocol.rs — Studio Hub 通信协议

## 概述

`hub_protocol.rs` 定义了 Makepad Studio 中 Hub 进程与各客户端（如文件浏览器、构建系统、终端、AI agent）之间通信的完整协议。Hub 是一个中心化路由进程，管理文件系统、构建任务、终端会话、AI agent 和日志查询。

所有类型均通过 `makepad_micro_serde` 的 `SerBin/DeBin/SerJson/DeJson` 派生支持序列化。

---

## 核心标识符

### ClientId
```rust
pub struct ClientId(pub u16);   // 客户端标识（最多 65535 个并发客户端）
```

### QueryId
```rust
pub struct QueryId(pub u64);    // 查询/请求标识
```

`QueryId` 采用 lane 编码：每个 `ClientId` 映射到 0-15 之间的 lane，`counter` 递增时乘以 16 后加上 lane 值。这样可以从 `QueryId` 反向推导出 `client_id`：
```rust
lanes = 16
client_id = query_id % 16
counter   = query_id / 16
```

### ClientToHubEnvelope
```rust
pub struct ClientToHubEnvelope {
    pub query_id: QueryId,
    pub msg: ClientToHub,
}
```

---

## ClientToHub（客户端 → Hub 请求）

### 文件系统

| 变体 | 说明 |
|------|------|
| `LoadFileTree { mount: String }` | 加载 mount 的文件树 |
| `OpenTextFile { path: String }` | 在编辑器中打开文本文件 |
| `SaveTextFile { path: String, content: String }` | 保存文本文件 |
| `DeleteFile { path: String }` | 删除文件 |
| `ReadTextFile { path: String }` | 读取完整文件内容 |
| `ReadTextRange { path: String, start_line: usize, end_line: usize }` | 读取文件的行范围 |
| `FindFiles { mount, pattern, is_regex, max_results }` | 查找文件（仅文件名匹配） |

### Mount 与分支管理

| 变体 | 说明 |
|------|------|
| `Mount { name: String, path: String }` | 挂载一个目录（作为项目 mount） |
| `Unmount { name: String }` | 卸载 mount |
| `ObserveMount { mount: String, primary: Option<bool> }` | 观察 mount 的变化 |
| `CreateBranch { mount, name, from_ref }` | 创建 Git 分支 |
| `DeleteBranch { mount, name }` | 删除 Git 分支 |
| `GitLog { mount, max_count }` | 获取 Git 提交历史 |

### 构建控制

| 变体 | 说明 |
|------|------|
| `ListBuilds` | 列出所有活跃构建 |
| `ListAppSockets` | 列出所有应用 WebSocket 连接 |
| `RunItem { mount, name }` | 通过 runnable item 运行应用 |
| `Cargo { mount, args, env, buildbox }` | 执行 cargo 命令 |
| `Run { mount, process, args, standalone, env, buildbox }` | 运行任意进程 |
| `StopBuild { build_id }` | 停止构建 |
| `ClearBuild { build_id }` | 清除构建的 UI 标签页 |

### 应用交互

| 变体 | 说明 |
|------|------|
| `ForwardToApp { build_id, msg_bin }` | 透传原始字节到应用 |
| `TypeText { build_id, text }` | 在应用中输入文本 |
| `Return { build_id, auto_dump }` | 发送回车键 |
| `Click { build_id, x, y }` | 点击指定坐标 |
| `Screenshot { build_id, kind_id }` | 请求截图 |
| `WidgetTreeDump { build_id }` | 请求 widget 树 |
| `WidgetQuery { build_id, query }` | 查询 widget |
| `WidgetSnapshot { build_id }` | 请求完整 widget 快照 |
| `RunViewInput { build_id, window_id, msg_bin }` | 向 RunView 窗口发送输入 |
| `RunViewResize { build_id, window_id, width, height, dpi }` | 调整 RunView 窗口大小 |

### 终端

| 变体 | 说明 |
|------|------|
| `TerminalOpen { path, cols, rows, env }` | 打开终端 |
| `TerminalInput { path, data }` | 向终端发送输入数据 |
| `TerminalViewportRequest { path, cols, rows, pty_rows, top_row }` | 请求终端视口更新 |
| `TerminalClose { path }` | 关闭终端 |

### AI

| 变体 | 说明 |
|------|------|
| `AiGetState { mount }` | 获取 mount 的 AI 状态 |
| `AiCreateAgent { mount, title }` | 创建 AI agent |
| `AiDeleteAgent { mount, agent_id }` | 删除 AI agent |
| `AiSelectAgent { mount, agent_id }` | 选择当前 AI agent |
| `AiSetBackend { mount, backend_id }` | 设置 AI 后端 |
| `AiSendPrompt { mount, agent_id, text }` | 发送提示词 |
| `AiCancelPrompt { mount, agent_id }` | 取消 AI 响应 |

### 搜索与查询

| 变体 | 说明 |
|------|------|
| `SearchFiles { mount, pattern, is_regex, glob, max_results }` | 在文件内容中搜索（同 `FindInFiles`） |
| `FindInFiles { mount, pattern, is_regex, glob, max_results }` | 在文件内容中搜索（同上） |
| `QueryLogs { build_id, level, source, file, pattern, is_regex, since_index, live }` | 查询日志 |
| `QueryProfiler { build_id, sample_type, time_start, time_end, max_samples, live }` | 查询性能样本 |
| `CancelQuery { query_id }` | 取消正在进行的查询 |

注：`SearchFiles` 和 `FindInFiles` 具有完全相同的字段。两者都在协议中保留以支持不同客户端。

### 构建盒管理

| 变体 | 说明 |
|------|------|
| `ListBuildBoxes` | 列出所有 BuildBox |
| `BuildBoxSyncNow { name }` | 立即同步 BuildBox |

### 脚本 CI

| 变体 | 说明 |
|------|------|
| `RunScriptTask { script_path }` | 运行脚本任务 |
| `StopScriptTask { task_id }` | 停止脚本任务 |
| `ListScriptTasks` | 列出所有脚本任务 |

### 日志

| 变体 | 说明 |
|------|------|
| `LogClear` | 清除所有日志 |

---

## HubToClient（Hub → 客户端响应）

### 连接

| 变体 | 说明 |
|------|------|
| `Hello { client_id }` | 连接建立，分配客户端 ID |

### 文件系统

| 变体 | 说明 |
|------|------|
| `FileTree { mount, data }` | 文件树数据 |
| `FileTreeDiff { mount, changes }` | 文件树增量更新 |
| `TextFileOpened { path, content, git_status, line, column }` | 文件已打开 |
| `TextFileRead { path, content }` | 文件读取结果 |
| `TextFileRange { path, start_line, end_line, total_lines, content }` | 文件范围读取结果 |
| `TextFileSaved { path, result }` | 文件保存结果 |
| `FileChanged { path }` | 文件已更改通知 |
| `FindFileResults { query_id, paths, done }` | 文件查找结果 |
| `GitLog { mount, log }` | Git 日志结果 |

### 构建

| 变体 | 说明 |
|------|------|
| `Builds { builds }` | 当前活跃构建列表 |
| `AppSockets { sockets }` | 应用 WebSocket 列表 |
| `RunItems { mount, items }` | mount 的 runnable items 列表 |
| `BuildStarted { build_id, mount, package }` | 构建已开始 |
| `BuildStopped { build_id, exit_code }` | 构建已停止 |
| `BuildCleared { build_id }` | 构建标签页已清除 |
| `AppStarted { build_id }` | 应用已启动 |

### 应用交互

| 变体 | 说明 |
|------|------|
| `Screenshot { query_id, build_id, kind_id, path, width, height }` | 截图结果（文件路径 + 尺寸） |
| `WidgetTreeDump { query_id, build_id, dump }` | Widget 树文本转储 |
| `WidgetQuery { query_id, build_id, query, rects }` | Widget 查询结果（矩形列表） |
| `WidgetSnapshot { query_id, build_id, widgets }` | Widget 快照结果 |

### RunView

| 变体 | 说明 |
|------|------|
| `RunViewCreated { build_id, window_id }` | RunView 窗口已创建 |
| `RunViewSwapchain { build_id, window_id, swapchain_desc }` | RunView 交换链描述 |
| `RunViewFrame { build_id, window_id, frame_id, width, height, codec, data }` | RunView 帧数据 |
| `RunViewDrawComplete { build_id, window_id, presentable_draw }` | RunView 绘制完成 |
| `RunViewCursor { build_id, cursor }` | RunView 光标变化 |
| `RunViewInputViz { build_id, kind, x, y }` | RunView 输入可视化 |
| `RunViewKeyFocusRect { build_id, x, y, width, height }` | RunView 键盘焦点矩形 |
| `RunViewDestroyed { build_id, window_id }` | RunView 窗口已销毁 |

### 终端

| 变体 | 说明 |
|------|------|
| `TerminalOpened { path }` | 终端已打开 |
| `TerminalFramebuffer { path, frame }` | 终端帧缓冲更新 |
| `TerminalTitle { path, title }` | 终端标题变化 |
| `TerminalExited { path, code }` | 终端进程已退出 |

### AI

| 变体 | 说明 |
|------|------|
| `AiMountState { mount, state }` | AI mount 状态更新 |

### 搜索与查询

| 变体 | 说明 |
|------|------|
| `SearchFileResults { query_id, results, done }` | 文件搜索逐批结果 |
| `QueryLogResults { query_id, entries, done }` | 日志查询结果 |
| `QueryProfilerResults { query_id, event_samples, gpu_samples, gc_samples, total_in_window, done }` | 性能分析结果 |
| `QueryCancelled { query_id }` | 查询已取消 |

### BuildBox

| 变体 | 说明 |
|------|------|
| `BuildBoxes { boxes }` | BuildBox 列表 |
| `BuildBoxConnected { info }` | BuildBox 已连接 |
| `BuildBoxDisconnected { name }` | BuildBox 已断开 |

### 脚本 CI

| 变体 | 说明 |
|------|------|
| `ScriptTasks { tasks }` | 脚本任务列表 |
| `ScriptTaskStarted { task_id, script_path }` | 任务已开始 |
| `ScriptTaskOutput { task_id, build_id, message, level }` | 任务输出 |
| `ScriptTaskResult { task_id, status, attachments }` | 任务完成结果 |

### 日志

| 变体 | 说明 |
|------|------|
| `LogCleared` | 日志已清除 |

### 错误

| 变体 | 说明 |
|------|------|
| `Error { message }` | 通用错误消息 |

---

## 共享数据类型

### 枚举

| 类型 | 变体 |
|------|------|
| `RunViewInputVizKind` | `ClickDown`, `ClickUp`, `TypeText`, `Return` |
| `FrameCodec` | `ZstdRgba`, `Jpeg`, `Png` |
| `FileNodeType` | `File`, `Dir` |
| `GitStatus` | `Clean`, `Modified`, `Staged`, `Added`, `Untracked`, `Deleted`, `Conflict`, `Ignored`, `Unknown` |
| `FileTreeChange` | `Added`, `Removed`, `Modified` |
| `FileError` | `NotFound`, `InvalidPath`, `Io`, `Git`, `Other` |
| `SaveResult` | `Ok`, `Err(FileError)` |
| `AiMessageRole` | `User`, `Assistant`, `Thinking`, `System`, `ToolCall`, `ToolResult`, `Error` |
| `BuildBoxStatus` | `Idle`, `Syncing`, `Building`, `Offline` |
| `TaskStatus` | `Running`, `Passed`, `Failed`, `Warned`, `Cancelled` |
| `LogSource` | `Cargo`, `ChildApp`, `BuildBox`, `Studio`, `Terminal`, `ScriptCi`, `Other(LiveId)` |
| `DeltaKind` | `Write`, `Delete`, `MkDir` |

### 结构体

| 类型 | 关键字段 |
|------|----------|
| `FileTreeData` | `nodes: Vec<FileNode>` |
| `FileNode` | `path, name, node_type, git_status` |
| `GitLog` | `commits: Vec<GitCommitInfo>` |
| `GitCommitInfo` | `hash, message, author, timestamp` |
| `BuildInfo` | `build_id, mount, package, active` |
| `AppSocketInfo` | `web_socket_id, build_id, crate_name, mount, package, build_active` |
| `RunItem` | `name, in_studio` |
| `AiAgentId` | `(u64)` 新类型 |
| `AiMessage` | `role, text` |
| `AiBackendInfo` | `id, label, detail, configured` |
| `AiAgentSummary` | `agent_id, title, backend_id, status, pending, updated_at, message_count` |
| `AiAgentState` | `agent_id, title, backend_id, status, pending, messages` |
| `AiMountState` | `backends, active_backend_id, active_agent_id, agents, active_agent, live_markdown` |
| `BuildBoxInfo` | `name, platform, arch, status` |
| `ScriptTaskInfo` | `task_id, script_path, status, started_at` |
| `Attachment` | `name, data, mime, build_id` |
| `SearchResult` | `path, line, column, line_text` |
| `LogEntry` | `index, timestamp, build_id, level, source, message, file_name, line, column` |
| `EventSample` | `at, label, event_u32, event_meta, start, end` |
| `GPUSample` | `at, label, start, end, draw_calls, instances, vertices, instance_bytes, uniform_bytes, vertex_buffer_bytes, texture_bytes` |
| `GCSample` | `at, label, start, end, heap_live` |
| `TerminalCellDiff` | `changed: Vec<TerminalCellUpdate>` |
| `TerminalCellUpdate` | `x, y, ch` |
| `TerminalFramebuffer` | `frame_id, cols, rows, top_row, total_lines, cursor_col, cursor_row, cursor_visible, default_fg_rgb, default_bg_rgb, bracketed_paste, cursor_keys_application_mode, is_tui, cells` |
| `FileDelta` | `path, kind` |
| `FileHash` | `path, size, mtime_ns, mode, is_symlink, symlink_target, content_blake3` |

---

## BuildBox 子协议

### HubToBuildBox (Hub → BuildBox)
```rust
pub enum HubToBuildBox {
    TreeHash { hash: String },         // 设置树哈希
    SyncFiles { files: Vec<FileDelta> }, // 同步文件差异
    RequestTreeHash,                   // 请求树哈希
    CargoBuild { build_id, mount, args, env }, // 执行 cargo 构建
    StopBuild { build_id },            // 停止构建
    Ping,                              // 心跳
}
pub struct HubToBuildBoxVec(pub Vec<HubToBuildBox>);
```

### BuildBoxToHub (BuildBox → Hub)
```rust
pub enum BuildBoxToHub {
    Hello { name, platform, arch, tree_hash },
    FileHashes { files: Vec<FileHash> },
    SyncComplete { tree_hash },
    SyncError { error: String },
    BuildOutput { build_id, line: String },
    BuildStarted { build_id },
    BuildStopped { build_id, exit_code },
    Pong,
}
pub struct BuildBoxToHubVec(pub Vec<BuildBoxToHub>);
```
