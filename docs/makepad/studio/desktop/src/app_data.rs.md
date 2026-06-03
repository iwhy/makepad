# `app_data.rs` — Studio Desktop 数据结构定义

## 模块作用

定义应用运行时的所有数据结构，包括标签页状态、日志条目、文件树、mount 配置等。这些数据结构在 `App.data` 中作为全局状态存在。

## 标签页状态结构体

### `RunTabState`
```rust
pub struct RunTabState {
    pub mount: String,        // 所属 mount
    pub package: String,      // 包名
    pub build_id: QueryId,    // 构建 ID
    pub status: String,       // 状态文本
    pub window_id: Option<usize>, // 运行窗口 ID
}
```

### `LogTabState`
```rust
pub struct LogTabState {
    pub mount: String,
    pub build_id: QueryId,
}
```

### `ProfilerTabState`
```rust
pub struct ProfilerTabState {
    pub mount: String,
    pub build_id: QueryId,
    pub title: String,
}
```

## UI 日志结构体

### `UiLogLocation`
日志位置信息，包含虚拟路径、行号和列号。提供 `display_label()` 方法生成 `path:line:column` 格式。

### `UiLogEntry`
```rust
pub struct UiLogEntry {
    pub level: LogLevel,         // 日志级别
    pub source: LogSource,       // 日志来源
    pub message: String,         // 消息文本
    pub location: Option<UiLogLocation>,  // 提取的文件位置
}
```

### `UiProfilerSamples`
性能采样数据容器：
- `event_samples: Vec<EventSample>` — 事件采样
- `gpu_samples: Vec<GPUSample>` — GPU 采样
- `gc_samples: Vec<GCSample>` — GC 采样
- `total_in_window: usize` — 窗口内总采样数

## `MountState`（每个 mount 的独立状态）

包含字段：
| 字段 | 类型 | 用途 |
|------|------|------|
| `root` | `PathBuf` | 挂载根目录路径 |
| `tab_id` | `Option<LiveId>` | 顶层 mount tab 的 ID |
| `sidebar_restore_width` | `Option<f64>` | 侧边栏折叠前宽度 |
| `bottom_panel_restore_height` | `Option<f64>` | 底部面板折叠前高度 |
| `file_tree_data` | `Option<FileTreeData>` | 文件树数据（来自 hub） |
| `run_items` | `Vec<RunItem>` | 可运行列表 |
| `log_entries` | `VecDeque<UiLogEntry>` | 全部日志条目（非 build 特定） |
| `terminal_files` | `Vec<String>` | 终端文件路径列表 |
| `terminals_initialized` | `bool` | 是否已完成初始化 |
| `terminal_path_to_tab` / `terminal_tab_to_path` | `HashMap` | 终端路径↔Tab 双向映射 |
| `ai_state` | `Option<AiMountState>` | AI manager 状态 |
| `file_filter` / `file_filter_results` / `file_filter_query` / `file_filter_pending` | 文件过滤相关 |
| `log_filter` / `log_tail` | 日志过滤和自动跟随 |

默认值：`log_tail = true`，其余字段均为空/None。

## `AppData`（全局应用状态）

包含大量 `HashMap` 保存各类映射关系：

**连接与挂载**：
- `studio: Option<HubConnection>` — hub 连接
- `mounts: HashMap<String, MountState>` — 所有 mount 状态
- `tab_to_mount: HashMap<LiveId, String>` — tab→mount 映射
- `active_mount: Option<String>` — 当前活动 mount

**文件树**：
- `file_tree: FlatFileTree` — 拍平后的文件树

**编辑器**：
- `sessions: HashMap<LiveId, CodeSession>` — 编辑器 session
- `path_to_tab` / `tab_to_path` — 路径↔tab 双向映射
- `pending_open_paths` / `pending_reload_paths` — 待处理文件
- `current_file_path: Option<String>` — 当前打开文件

**运行/日志/分析标签**：
- `run_tab_state` / `run_tab_by_build` — 运行标签
- `log_tab_state` / `log_tab_by_build` — 日志标签
- `profiler_tab_state` / `profiler_tab_by_build` — 分析标签

**数据存储**：
- `build_log_entries: HashMap<QueryId, VecDeque<UiLogEntry>>` — 构建日志
- `profiler_samples_by_build` — 性能采样
- `profiler_running_by_build` — 采样运行状态
- `build_to_mount` / `build_package` — 构建元数据

**日志/查询**：
- `active_log_build_by_mount: HashMap<String, QueryId>` — 当前日志 build
- `live_log_query: Option<QueryId>` — 实时日志查询
- `live_profiler_query_by_build` — 实时 profiler 查询
- `pending_log_jumps: HashMap<String, (usize, usize)>` — 待处理跳转

**终端**：
- `terminal_framebuffer_by_path` — 终端帧缓冲
- `terminal_frame_id_by_path` — 帧 ID 去重
- `terminal_open_paths` — 已打开终端
- `terminal_title_by_path` — 终端标题

**过滤/拖拽**：
- `file_filter_mount_by_query: HashMap<QueryId, String>`
- `pending_stop_all_mount: Option<String>`
- `run_panel_split_restore: HashMap<String, SplitterAlign>`

## `FlatFileTree`（拍平文件树）

将层级化 `FileTreeData` 转换为扁平的 ID 映射，便于快速查找和绘制。

### 内部结构
```rust
struct FlatNode {
    id: LiveId,
    path: String,
    name: String,
    node_type: FileNodeType,
    git_status: GitStatus,
    children: Vec<LiveId>,
}
```

### 主要方法

**`rebuild(&mut self, data: &FileTreeData)`**：
1. 过滤隐藏路径（`is_hidden_virtual_path` 检测以 `.` 开头的路径段）
2. 为每个节点创建 `FlatNode` 并建立 `path_to_id` 索引
3. 构建父子关系 —— 通过 `rsplit_once('/')` 找到父路径
4. 调用 `cascade_git_status_to_folders` 将子节点 git 状态向上传播
5. 排序：文件夹优先 → 按名称字母序

**`draw`**：从根节点开始递归绘制文件树。

**`git_status_dot_for_path`**：查询路径的 git 状态指示点。

**`cascade_git_status_to_folders`**：按深度从叶子到根排序，对每个文件夹节点，遍历子节点 git 状态并合并出最高优先级的状态。

### Git 状态优先级
```
Conflict(6) > Deleted(5) > Modified(4) > Staged(3) > Added(2) > Untracked(1) > Clean/Ignored/Unknown(0)
```

### GitStatusDotKind 映射
- `Added | Untracked` → `New`（绿色）
- `Modified | Staged` → `Modified`（黄色）
- `Deleted` → `Deleted`（红色）
- `Conflict` → `Mixed`
- `Clean | Ignored | Unknown` → `None`

## 辅助函数

### `is_hidden_virtual_path`
检测虚拟路径中是否包含以 `.` 开头的路径段（隐藏文件/目录）。

### `git_status_dot` / `merge_git_status` / `git_status_rank`
Git 状态到显示颜色的映射、合并规则和优先级排序。

## 单元测试

### `FlatFileTree` 测试
- 文件夹 git 状态从子节点正确级联
- Deleted 状态优先级高于 Modified
- stale 文件夹状态会被子节点正确覆盖

### `retain_persistable_editor_tabs` 测试
- 终端文件路径不被持久化
