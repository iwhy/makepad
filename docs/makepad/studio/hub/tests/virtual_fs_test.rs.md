# virtual_fs_test.rs

## 概述

测试 `VirtualFs` 虚拟文件系统的核心功能：mount 解析、分支路径（branch path）支持、文件树加载、Git 状态映射和 `.makepad` 目录过滤。

## 测试场景

### `resolves_mount_and_branch_paths`

- **场景**: 验证 VirtualFs 能正确解析 mount 根路径、分支路径（`@branch-name`）和分支文件的路径。
- **流程**:
  1. 创建临时目录，包含 `src/lib.rs` 和 `branch/feature-ui/src/lib.rs`。
  2. 挂载 `"makepad"` → `dir.path()`。
  3. 验证 `resolve_mount("makepad")` 返回 mount 根路径。
  4. 验证 `resolve_mount("makepad/@feature-ui")` 返回 `branch/feature-ui` 路径。
  5. 验证 `resolve_path("makepad/@feature-ui/src/lib.rs")` 路径解析正确。
  6. 加载文件树，验证 mount 根、分支节点和分支内文件都在 `nodes` 中。
- **验证重点**: 分支路径解析语义——`@branch-name` 映射到 `branch/branch-name/`。

### `git_statuses_are_mapped_for_tree_nodes`

- **场景**: 验证文件树节点包含正确的 Git 状态（`Modified`、`Untracked`）。
- **流程**:
  1. 创建 Git 仓库，提交 `tracked.txt`，然后修改它并创建新文件 `new_untracked.txt`。
  2. 挂载并加载文件树。
  3. 验证根节点 `git_status` 为 `Modified`。
  4. 验证 `tracked.txt` 状态为 `Modified`。
  5. 验证 `new_untracked.txt` 状态为 `Untracked`。
- **验证重点**: `VirtualFs` 集成了 `git status` 扫描，每个 `FileTreeNode` 携带 `git_status`。
- **注意**: 如果没有 `git` 命令则跳过测试。

### `load_file_tree_is_scoped_to_requested_mount`

- **场景**: 验证文件树加载结果仅包含指定 mount 下的节点，不混入其他 mount 的数据。
- **流程**:
  1. 创建两个 mount：`"alpha"` 含 `src/a.rs`，`"beta"` 含 `src/b.rs`。
  2. 加载 `"alpha"` 的文件树，断言所有节点路径以 `"alpha"` 开头，不包含 `"beta"` 路径，包含 `"alpha/src/a.rs"`。
  3. 对称验证 `"beta"`。
- **验证重点**: Mount 隔离性——不同 mount 的文件树互不干扰。

### `load_file_tree_ignores_makepad_state_directory`

- **场景**: 验证文件树忽略 `.makepad` 目录（状态目录，不暴露给 UI）。
- **流程**:
  1. 创建 `.makepad/ai_chats/chat.json` 和 `src/lib.rs`。
  2. 加载文件树，验证 `src/lib.rs` 存在，而 `.makepad` 及其内容不在节点列表中。
- **验证重点**: 内部 `.makepad` 目录的过滤逻辑。

## 测试模式

- **纯单元测试**：直接操作 `VirtualFs`，不启动 Hub 后端。
- **Git 集成测试**：`git_status` 测试需要真实 Git 仓库，跳过条件 `!git_available()`。
- **隔离验证**：`tempdir` 自动清理临时文件。
