# file_tree_smoke.rs

## 概述

端到端冒烟测试，验证通过 in-process 连接请求文件树（`LoadFileTree`）并接收 `FileTree` 响应的完整流程。

## 测试场景

### `load_file_tree_smoke`

- **场景**: 启动 Hub 后端并通过 `HubConnection` 发送 `LoadFileTree` 请求，验证收到正确的 `FileTree` 响应。
- **流程**:
  1. 创建临时目录作为 mount 根路径。
  2. 配置 `HubConfig`（一个 mount 名为 `"repo"`），`enable_in_process_gateway: false`。
  3. 通过 `StudioHub::start_in_process` 启动后端并获取 `HubConnection`。
  4. 发送 `ClientToHub::LoadFileTree { mount: "repo" }`。
  5. 在 2 秒超时内轮询 `connection.recv_timeout`，匹配 `HubToClient::FileTree` 消息。
  6. 断言 `mount` 为 `"repo"`，`data.nodes` 非空。
- **验证重点**:
  - In-process 连接的正确建立。
  - `LoadFileTree` → `FileTree` 请求-响应链路。
  - 文件树数据不为空（至少包含 mount 根节点）。

## 测试模式

- **冒烟测试**：仅验证基本流程可用。
- **轮询等待模式**：使用超时循环轮询 `recv_timeout` 等待特定消息。
- **in-process 模式**：后端在当前进程中启动，通过内存通道通信而非 WebSocket。
