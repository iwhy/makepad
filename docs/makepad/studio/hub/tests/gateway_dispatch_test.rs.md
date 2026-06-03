# gateway_dispatch_test.rs

## 概述

测试 Gateway（WebSocket 网关）的 UI 客户端连接完整流程：WebSocket 握手、`Hello` 消息分配、`LoadFileTree` 请求/响应路由。

## 测试辅助函数

- `find_free_port()`: 获取一个可用端口。
- `wait_for_event(runtime, timeout, matcher)`: 轮询 `NetworkRuntime` 事件。
- `wait_for_ws_binary(runtime, socket_id, timeout)`: 等待并返回指定 socket 的下一个二进制消息。

## 测试场景

### `websocket_ui_hello_and_load_file_tree_roundtrip`

- **场景**: 模拟 UI 客户端通过 WebSocket 连接到 headless 后端，完成握手和文件树请求。
- **流程**:
  1. 创建临时项目目录，启动 headless 后端。
  2. `NetworkRuntime` 打开 WebSocket 连接到 `ws://127.0.0.1:{port}/ui`。
  3. 等待 `WsOpened` 确认连接建立。
  4. 接收第一个二进制消息并解码为 `HubToClient::Hello`，获取 `client_id`（验证不为 `u16::MAX`）。
  5. 发送 `ClientToHubEnvelope`（`LoadFileTree` 请求）。
  6. 接收响应并解码为 `HubToClient::FileTree`，断言 `mount == "repo"` 且节点包含 `"repo/src/lib.rs"`。
  7. 关闭 socket。
- **验证重点**:
  - Gateway 的 `/ui` 路径接受 WebSocket 连接。
  - 新客户端自动收到 `Hello` 消息（分配 client_id）。
  - 请求-响应消息路由正确（envelope 中的 `query_id` 匹配）。

## 测试模式

- **WebSocket 集成测试**：使用 `NetworkRuntime` 模拟 WebSocket 客户端。
- **无 in-process gateway**：`enable_in_process_gateway: false` 确保走完整网络栈。
- **端口容错**: 如果端口绑定失败则优雅跳过（`find_free_port` 可处理竞态）。
