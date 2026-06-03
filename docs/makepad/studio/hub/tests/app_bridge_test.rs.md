# app_bridge_test.rs

## 概述

测试 App Bridge（应用桥接）的完整消息转发流程：UI 客户端通过 Hub 向运行中的应用发送请求（WidgetTreeDump、WidgetQuery、WidgetSnapshot），应用响应后 Hub 将结果路由回 UI 客户端。

## 测试辅助函数

- `find_free_port()`: 获取可用端口。
- `wait_for_event(runtime, timeout, matcher)`: 轮询事件。
- `wait_for_ws_binary(runtime, socket_id, timeout)`: 等待二进制消息。
- `wait_for_app_socket_registration(runtime, ui_socket, client_id, build_id)`: 轮询 `ListAppSockets` 直到应用 socket 注册成功。

## 测试场景

### `websocket_app_bridge_widget_dump_roundtrip`

- **场景**: UI 请求 `WidgetTreeDump`，Hub 转发到 App，App 响应后 Hub 转发回 UI。
- **流程**:
  1. 启动 headless 后端。
  2. UI 客户端连接 `/ui` → 收到 `Hello`。
  3. App 客户端连接 `/app?build={build_id}&crate=makepad-example-xr`。
  4. 轮询 `ListAppSockets` 直到 App socket 注册。
  5. UI 发送 `WidgetTreeDump { build_id }`。
  6. App socket 收到 `StudioToApp::WidgetTreeDump`（request_id 匹配）。
  7. App 回复 `AppToStudio::WidgetTreeDump(WidgetTreeDumpResponse)`。
  8. UI socket 收到 `HubToClient::WidgetTreeDump`（query_id 和 build_id 匹配，dump 内容包含 "root"）。

### `websocket_app_bridge_widget_query_roundtrip`

- **场景**: 类似流程，测试 `WidgetQuery` 消息。
- **流程**:
  1-4. 同上。
  5. UI 发送 `WidgetQuery { build_id, query: "id:math_tab" }`。
  6. App socket 收到 `StudioToApp::WidgetQuery`（request_id 和 query 匹配）。
  7. App 回复 `WidgetQueryResponse`（包含 rects）。
  8. UI 收到 `HubToClient::WidgetQuery`（query 和 rects 匹配）。

### `websocket_app_bridge_widget_snapshot_roundtrip`

- **场景**: 测试 `WidgetSnapshot` 消息。
- **流程**:
  1-4. 同上。
  5. UI 发送 `WidgetSnapshot { build_id }`。
  6. App socket 收到 `StudioToApp::WidgetSnapshot`（request_id 匹配）。
  7. App 回复 `WidgetSnapshotResponse`（包含 widget 列表——id、类型、位置、大小、值等）。
  8. UI 收到 `HubToClient::WidgetSnapshot`（widget 列表完全匹配）。

## 测试模式

- **三角色集成测试**: UI 客户端 + Hub + App 客户端，通过 WebSocket 通信。
- **App socket 注册轮询**: 使用 `ListAppSockets` 主动轮询，确保应用 socket 在 Hub 中注册完毕。
- **消息转发代理**: Hub 在 UI 和 App 之间透明转发消息，不修改 payload。
- **`StudioToAppVec` / `AppToStudioVec`**: App 侧的协议使用 `Vec` 包裹，支持批量消息。
