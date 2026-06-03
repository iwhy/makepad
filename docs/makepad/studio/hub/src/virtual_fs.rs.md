# `virtual_fs.rs` — 虚拟文件系统和挂载机制

## 文件位置
- 路径: `studio/hub/src/virtual_fs.rs`
- 行数: 1331 行
- 作用: 提供虚拟文件路径到真实文件系统的映射，支持多挂载点、git 分支、文件搜索、Rabin-Karp/正则内容搜索

## 核心数据结构

### `MountPoint`
```rust
pub struct MountPoint {
    pub name: String,    // 挂载名，如 "makepad"
    pub path: PathBuf,   // 真实路径，如 /home/user/repo
}
```

### `VirtualFs`
```rust
pub struct VirtualFs {
    mounts: HashMap<String, MountPoint>,
    open_buffers: HashMap<String, String>,  // 虚拟路径 → 文件内容缓存
}
```
- `open_buffers` 保存已打开的文本文件内容，支持多次读取免 IO

### `VirtualFsError`
```rust
pub enum VirtualFsError {
    MissingMount(String),
    InvalidVirtualPath(String),
    Search(String),
    Io(std::io::Error),
    Git(String),
}
```
- 实现 `Display`、`Error`、`From<io::Error>`
- 可转换为协议类型 `FileError`

### `StatusContext`
```rust
struct StatusContext {
    in_repo: bool,
    status_map: HashMap<String, GitStatus>,
}
```
内部辅助结构，缓存 git 仓库的文件状态，避免在遍历时重复查询。

### `SearchFileCandidate`
```rust
struct SearchFileCandidate {
    index: usize,
    path: PathBuf,
    virtual_path: String,
}
```
用于线程池并行搜索的文件候选。

### `FindInFilesMatcher`
```rust
enum FindInFilesMatcher {
    Literal(Vec<u8>),
    Regex(Regex),
}
```

## 虚拟路径格式

路径格式为 `{mount_name}[/{subpath}]`，分支路径格式为 `{mount_name}/@{branch_name}/{subpath}`。

### 路径解析

| 方法 | 说明 |
|------|------|
| `split_mount_and_rest(input)` | 拆分 `"mount/path"` → `("mount", "path")`；无斜杠返回 `("name", "")` |
| `split_head_tail(input)` | 拆分为第一个 `/` 前后的两部分 |
| `parse_branch_segment(segment)` | 去掉 `@` 前缀后 percent-decode |
| `resolve_path(path)` | 完整路径解析：提取 mount → 处理 `@branch` → 拼接真实路径 |
| `resolve_mount(mount)` | 解析 mount 根路径，可包含 `@branch`（内部处理） |

## 核心方法

### 文件操作

| 方法 | 说明 |
|------|------|
| `mount(name, path)` | 注册挂载点，path 会自动 `canonicalize` |
| `unmount(name)` | 移除挂载点 |
| `mounts()` | 返回按名称排序的所有挂载点 |
| `read_text_file(path)` | 读取文件到字符串并缓存到 `open_buffers` |
| `read_text_range(path, start, end)` | 读取指定行范围（1-indexed），返回 `(content, total_lines)` |
| `open_text_file(path)` | 同 `read_text_file`，显式打开 |
| `save_text_file(path, content)` | 写入文件（自动创建父目录），更新缓存 |
| `delete_path(path)` | 删除文件或目录，清除缓存 |

### Git 分支操作

| 方法 | 说明 |
|------|------|
| `create_branch(mount, name, from_ref)` | 创建 git 分支 + `branch/{name}` 目录（`local_clone_depth1` 浅克隆） |
| `delete_branch(mount, name)` | 删除 git 分支 + 对应的本地目录 |
| `git_log(mount, max_count)` | 获取 git 提交日志 |

`create_branch` 内部：解析 `from_ref`（先尝试直接 resolve，再尝试 `refs/heads/{name}`）→ 创建 git 分支 → 如果 `branch/{name}` 不存在则 shallow clone。

### 文件树遍历

`load_file_tree(mount) -> FileTreeData`:
1. 先加载 mount 根目录（挂载名作为虚拟根）
2. 对根目录调用 `load_status_context` 获取 git 状态
3. `walk_dir` 递归遍历（跳过 `.git`、`.makepad`、`branch`、`target`）
4. 扫描 `mount/branch/` 下的所有分支根目录
5. 对每个分支，以 `mount/@branch_name` 为虚拟根再次遍历

`walk_dir` 参数 `skip_branch_dir`：根级遍历时跳过 `branch/` 目录，分支级遍历时不需要跳过（因为分支根不包含 `branch/`）。

### 文件搜索

**`find_files(mount, pattern, max_results)`** — 按文件名搜索：
- DFS 遍历目录树，过滤 `.git` 和 `target`
- 使用 `virtual_path.contains(pattern)` 匹配
- 支持指定 mount 或搜索所有挂载点

**`find_in_files(mount, pattern, is_regex, glob, max_results, regex_search_pool)`** — 按内容搜索：
- Literal 模式：使用 makepad_rabin_karp 的 `search_with_limit`
- Regex 模式：支持线程池并行搜索
- glob 过滤：逗号分隔的通配符列表，支持 `*` 和 `?`

### 线程池并行 Regex 搜索

`search_files_with_regex_pool(files, pattern, max_results, pool)`:
1. 初始化 `remaining = AtomicUsize(max_results)` — 全局结果配额
2. 分发第一批（`in_flight_cap = worker_count * 2`）任务到线程池
3. 每个任务：`search_regex_file_with_budget` 读取文件 → 检查 `remaining` 配额 → 正则搜索
4. 使用 `try_claim_search_result_slot`（CAS compare_exchange_weak）原子扣减配额
5. 使用 `dispatch_regex_search_job` 分发任务
6. 结果按文件原始索引顺序组装

### Git 状态集成

`load_status_context(real_root)`:
- 打开 Git 仓库 → `status_for_file_tree()` → 将条目存入 `HashMap<String, GitStatus>`

`git_status_from_file_status(GitFileStatus) -> GitStatus`:
- 映射关系：Modified→Modified, Deleted→Deleted, Untracked→Untracked, Staged→Staged, StagedDeleted→Deleted, StagedNew→Added

`aggregate_root_git_status(ctx) -> GitStatus`:
- 优先级排序：Conflict > Deleted > Modified > Staged > Added > Untracked > Clean

## 辅助工具函数

| 函数 | 说明 |
|------|------|
| `percent_encode(input)` | URL 百分比编码（保留字母数字、`-`、`_`、`.`） |
| `percent_decode(input)` | URL 百分比解码 |
| `wildcard_match(pattern, text)` | shell glob 匹配（支持 `*` 和 `?`） |
| `should_search_virtual_path(path, vpath, glob)` | 无 glob 时默认只搜索 `.rs` `.md` `.toml` |
| `search_file_content(vpath, content, matcher, out, max)` | 单文件内容搜索 |
| `line_starts(bytes)` | 计算每行起始字节偏移 |
| `search_result_for_byte_offset(content, line_starts, vpath, offset)` | 从字节偏移构造 SearchResult |
| `compact_line_text(line, match_offset)` | 截断长行（最多 220 字节），匹配位置居中 |
| `next_char_boundary(text, pos)` | 跳到下一个 UTF-8 字符边界 |
| `scan_branch_roots(mount_root)` | 扫描 `branch/` 下的所有子目录 |
| `slash_rel(root, path)` | 计算相对路径并标准化路径分隔符为 `/` |
| `from_hex(b)`, `hex(v)` | 十六进制转换 |

## 设计观察

- 文件搜索默认限制在 `.rs`/`.md`/`.toml`，由 `should_search_virtual_path` 控制
- Branch 通过 `branch/{name}` 目录实现，而非 git worktree
- 线程池搜索使用 `AtomicUsize` 配额控制结果总数，不依赖通道容量
- open_buffers 在 `clone_for_search` 中被清空，避免搜索线程携带大缓存
