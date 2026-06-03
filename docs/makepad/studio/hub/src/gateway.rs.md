# `gateway.rs` — HTTP/WebSocket 网关

## 文件位置
- 路径: `studio/hub/src/gateway.rs`
- 行数: 341 行
- 作用: 提供 HTTP + WebSocket 网关，负责外部客户端、App 进程、BuildBox 的 WebSocket 连接管理和消息路由

## 数据结构

### `SocketRole`
```rust
enum SocketRole { Client, App, BuildBox }
```
标记每个 WebSocket 连接的角色。

### `GatewayHandle`
```rust
pub struct GatewayHandle {
    pub listen_address: SocketAddr,
    pub request_thread: JoinHandle<()>,
    pub http_thread: JoinHandle<()>,
}
```

### `AppConnectInfo`
```rust
struct AppConnectInfo {
    build_id: Option<QueryId>,
    crate_name: Option<String>,
}
```
用于解析 App 连接请求中的查询参数。

## `fn start_http_gateway(listen_address, post_max_size, event_tx)`

### 启动流程
1. 创建 `mpsc::channel<HttpServerRequest>` 作为请求通道
2. 调用 `start_http_server(HttpServer { ... })` 启动 HTTP 服务
3. 在 `request_thread` 中循环处理 `HttpServerRequest`

### WebSocket 连接处理 (`ConnectWebSocket`)

根据请求路径分配角色：

| 路径/路由 | 角色 | 说明 |
|-----------|------|------|
| `/ui` | `SocketRole::Client` | Studio UI 客户端（桌面），发送 `HubEvent::ClientConnected` |
| `/app/{build_id}` | `SocketRole::App` | 应用进程直连，路径中包含 build ID |
| `/app?build=N&crate=X` | `SocketRole::App` | 应用进程连接，查询参数模式 |
| `/$studio_buildbox` | `SocketRole::BuildBox` | 构建盒（远程构建器） |
| 其他路径 | 拒绝 | 发送 `Error` 消息后关闭连接 |

### WebSocket 断开处理 (`DisconnectWebSocket`)
根据角色发送对应的 `HubEvent`：
- `ClientDisconnected`
- `AppDisconnected`
- `BuildBoxDisconnected`

### 消息处理

| 请求类型 | Client | App | BuildBox |
|---------|--------|-----|----------|
| `BinaryMessage` | `ClientBinary` | `AppBinary` | `BuildBoxBinary` |
| `TextMessage` | `ClientText` | 忽略 | 忽略 |

### HTTP 请求
- `Get /$studio_health` → 返回 `200 OK`（健康检查）
- 其他 Get/Post → 返回 `404 Not Found`

## `fn parse_app_path(path) -> Option<AppConnectInfo>`

支持两种格式：
1. `/app/{build_id}` — 路径风格，如 `/app/42`
2. `/app?build=42&crate=makepad-example-xr` — 查询参数风格

规则：
- 路径风格中 `rest` 不允许包含额外 `/`
- 查询参数风格：`build` 必须是合法 u64，`crate` 可选
- 至少需要提供 `build_id` 或 `crate_name` 之一

## HTTP 响应辅助函数

| 函数 | 说明 |
|------|------|
| `ok_response(body, content_type)` | 返回 `200 OK`，带 `Cache-Control: no-cache` |
| `not_found_response()` | 返回 `404 Not Found` |

## 测试 (`#[cfg(test)]`)

| 测试 | 验证点 |
|------|--------|
| `parse_clean_app_path` | `/app/42` 和查询参数格式正确解析 |
| `reject_missing_or_invalid_build_id` | 空、非数字、多余路径段、无效 build、`/ui` 路径均拒绝 |

## 设计观察

- 使用原始二进制通道（`response_sender: Sender<Vec<u8>>`），协议层序列化由 dispatch 处理
- Client 角色额外接收 `TextMessage`（可能是 JSON 调试命令）
- `SocketRole` 从 `HashMap<u64, SocketRole>` 中删除即关闭连接
- 健康检查路径 `/$studio_health` 硬编码
