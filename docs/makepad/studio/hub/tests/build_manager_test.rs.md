# build_manager_test.rs

## 概述

测试 `BuildManager` 的进程管理和生命周期控制：启动 cargo 构建进程、捕获输出、处理进程退出，以及在 Unix 上通过进程组终止构建。

## 测试场景

### `build_manager_emits_output_and_exit_for_cargo`

- **场景**: 验证 `BuildManager::start_cargo_run` 能启动 cargo 子进程，捕获其标准输出，并在进程退出时发送 `ProcessExited` 事件。
- **流程**:
  1. 创建 `mpsc::channel` 接收 `HubEvent`。
  2. 初始化 `BuildManager` 和临时目录。
  3. 调用 `start_cargo_run(build_id, "repo", tmp, ["--version"], ...)`。
  4. 断言 `list_builds()` 长度为 1。
  5. 轮询 `rx` 通道：
     - 收到 `ProcessOutput`（包含构建输出行，包含 "cargo"）→ 标记 `saw_output`。
     - 收到 `ProcessExited`（exit_code = 0）→ 调用 `mark_exited`，标记 `saw_exit`，跳出。
  6. 断言 `saw_output` 和 `saw_exit` 均为 true。
  7. 断言 `list_builds()` 为空（mark_exited 后已清理）。
- **验证重点**: `start_cargo_run` 能在工作目录下执行 cargo，输出可通过 mpsc 通道获取，退出后 build 记录被移除。

### `build_manager_stop_build_kills_process_group` (Unix only)

- **场景**: 验证 `BuildManager::stop_build` 能终止整个进程组（包括后台子进程）。
- **流程**:
  1. 通过 `start_command_run` 启动一个 shell 脚本，该脚本启动 `sleep 30` 后台进程然后 `wait`。
  2. 轮询 `child.pid` 文件获取后台子进程 PID，验证其存活。
  3. 调用 `manager.stop_build(build_id)`。
  4. 轮询 `ProcessExited` 事件。
  5. 等待后台子进程消亡（`pid_exists` 返回 false）。
  6. 如果子进程未消亡则强制 `kill -9` 作为安全措施。
- **验证重点**:
  - `stop_build` 发送 SIGTERM 到整个进程组（`libc::kill(-pid, sig)`）。
  - 后台启动的子进程（`sleep 30`）也被一同终止。
  - 使用原始 `libc::kill` FFI 调用检查进程存活。
- **注意**: 仅在 Unix 系统上运行（`#[cfg(unix)]`），直接调用 `extern "C" { fn kill(pid: i32, sig: i32) -> i32; }`。

## 测试模式

- **单元测试**：直接测试 `BuildManager`，不启动 Hub 后端。
- **mpsc 通道**: 通过 `std::sync::mpsc::channel` 接收 `HubEvent`，模拟 Hub 调度层的事件消费。
- **进程组管理**: 使用 `start_command_run` 和进程组 ID 管理子进程。
- **安全清理**: 测试失败时也有 `kill_pid` 兜底清理，避免僵尸进程。
