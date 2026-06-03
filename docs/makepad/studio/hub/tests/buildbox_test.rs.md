# buildbox_test.rs

## 概述

测试 BuildBox（远程构建代理）的 WebSocket 通信全流程：buildbox 连接、注册、接收同步命令、执行远程 cargo 构建、日志查询和构建停止。

## 测试辅助函数

- `find_free_port()`: 绑定到 `127.0.0.1:0` 获取系统分配的可用端口。
- `wait_for_event(runtime, timeout, matcher)`: 轮询 `NetworkRuntime` 事件，直到匹配或超时。
- `wait_for_ui_message(runtime, socket_id, timeout, matcher)`: 从指定 WebSocket 接收 `HubToClient` 二进制消息，反序列化后匹配。
- `wait_for_buildbox_message(runtime, socket_id, timeout)`: 从 buildbox WebSocket 接收 `HubToBuildBoxVec` 消息。

## 测试场景

### `websocket_buildbox_remote_build_roundtrip`

- **场景**: 模拟完整的 buildbox 远程构建生命周期。
- **流程**:
  1. 创建临时项目目录（含 `src/lib.rs`），headless 后端启动。
  2. 客户端（UI）WebSocket 连接 → 收到 `Hello`，获取 `client_id`。
  3. BuildBox 代理 WebSocket 连接 → 发送 `BuildBoxToHub::Hello`（名称、平台、架构等）。
  4. UI 侧收到 `BuildBoxConnected` 通知。
  5. 客户端发起 `ListBuildBoxes` 查询 → 收到 `BuildBoxes` 列表，验证 linux buildbox 存在。
  6. 客户端发起 `BuildBoxSyncNow` → buildbox 侧收到 `RequestTreeHash` 命令。
  7. 客户端发起 `Cargo` 远程构建请求（指定 buildbox）→ buildbox 侧收到 `HubToBuildBox::CargoBuild` 命令，包含 `build_id`、`mount`、`args`。
  8. UI 侧收到 `BuildStarted` 通知。
  9. BuildBox 发送 `BuildOutput` → UI 侧 `QueryLogs` 能检索到该输出行。
  10. BuildBox 发送 `BuildStopped`（exit_code=0）→ UI 侧收到 `BuildStopped`。
- **验证重点**:
  - `ClientToHub::Cargo` 中的 `buildbox: Some("linux")` 路由到远程 buildbox。
  - BuildBox 连接的 `BuildBoxConnected` / `BuildBoxes` / `BuildBoxSyncNow` 消息流。
  - 构建输出通过 `BuildOutput` → `QueryLogs` 的正确传递。
  - 完整的远程构建 start → output → stop 生命周期。

## 测试模式

- **WebSocket 集成测试**：使用 `NetworkRuntime` 模拟多个 WebSocket 客户端。
- **多角色模拟**：同时扮演 UI 客户端和 BuildBox 代理。
- **超时轮询模式**：所有消息等待均使用 deadline 循环 + `recv_timeout`。
- **Graceful exit**: 测试结束时关闭所有 socket。
