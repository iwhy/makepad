# `hub.rs` — Studio Hub 入口和连接管理

## 文件位置
- 路径: `studio/hub/src/hub.rs`
- 行数: 321 行
- 作用: 定义 Hub 的配置结构、与客户端（进程内 UI 或外部）的连接管理、网关启动逻辑

## 常量

```rust
const IN_PROCESS_UI_CONNECTION_ID: u64 = 0;
```
进程内 UI（Studio 桌面端）使用固定的连接 ID `0`，WebSocket 网关的连接从 `1` 开始分配。

## 配置结构

### `MountConfig`
```rust
pub struct MountConfig {
    pub name: String,
    pub path: PathBuf,
}
```
定义虚拟文件系统的一个挂载点。

### `HubConfig`
```rust
pub struct HubConfig {
    pub listen_address: SocketAddr,  // 默认 127.0.0.1:8001
    pub post_max_size: u64,         // 默认 1MB
    pub mounts: Vec<MountConfig>,
    pub enable_in_process_gateway: bool,  // 默认 false
}
```
- Default 实现提供 `127.0.0.1:8001`、1MB post 大小

## 连接结构

### `HubHandle`
```rust
pub struct HubHandle {
    pub listen_address: SocketAddr,
    pub event_tx: Sender<HubEvent>,
    pub _gateway: GatewayHandle,
    pub _core_thread: JoinHandle<()>,
}
```
headless 模式（无 UI 进程）下返回的句柄，可通过 `event_tx` 发送事件。

### `HubConnection`
```rust
pub struct HubConnection {
    client_id: ClientId,
    web_socket_id: u64,
    event_tx: Sender<HubEvent>,
    recv_typed: ToUIReceiver<HubToClient>,
    next_counter: u64,
    gateway: Option<GatewayHandle>,
    _core_thread: JoinHandle<()>,
}
```

| 方法 | 说明 |
|------|------|
| `client_id()` | 返回分配的 ClientId |
| `studio_addr()` | 返回子进程回连地址字符串（如 `"127.0.0.1:8001"`） |
| `send(msg)` | 发送 `ClientToHub` 请求，返回分配的 `QueryId` |
| `cancel_query(query_id)` | 发送取消请求 |
| `try_recv()` | 非阻塞接收 `HubToClient` 响应 |
| `recv_timeout(timeout)` | 超时接收 |

`send()` 内部：
1. 用自增 `next_counter` 生成 `QueryId(client_id, seq)`
2. 包装为 `ClientToHubEnvelope`
3. 通过 `HubEvent::ClientEnvelope` 发送到事件通道
4. `wrapping_add` 防止溢出

## `StudioHub` — 启动入口

### `fn start_in_process(config) -> Result<HubConnection, String>`
进程内模式（Studio 桌面直接加载）：
1. 创建 `VirtualFs` 并挂载所有 mount
2. 可选启动 HTTP 网关（`enable_in_process_gateway`）
3. 创建 `HubCore` 并在独立线程中运行
4. 注册进程内客户端（`web_socket_id = 0`）
5. 等待 HubToClient::Hello，获取 `client_id`
6. 返回 `HubConnection`

### `fn start_headless(config) -> Result<HubHandle, String>`
Headless 模式（仅 CLI/后台）：
1. 创建 `VirtualFs` 并挂载
2. **必须**启动 HTTP 网关（供外部客户端连接）
3. 创建 `HubCore` 并在独立线程中运行
4. 返回 `HubHandle`（可通过 `event_tx` 直接发送事件）

## 端口回退机制

### `fn start_http_gateway_with_fallback(base, post_max_size, event_tx)`
- 尝试 `base` 到 `u16::MAX` 的每个端口
- 第一个成功绑定的端口即返回
- 全部失败则返回最后一个错误信息

### `fn gateway_bind_candidates(base) -> Vec<SocketAddr>`
- 如果端口为 0（临时端口），只尝试一次
- 否则生成 `[base.port..=u16::MAX]` 的候选列表

## 子进程地址计算

### `fn studio_local_addr_for_child(listen_address) -> String`
- 将 `0.0.0.0` 替换为 `127.0.0.1`
- 将 `::` 替换为 `::1`
- 显式 IP 则保持不变

### `fn studio_ext_addr_for_child(listen_address) -> String`
- 未指定 IP 时尝试 `detect_external_ipv4()` 获取公网 IP
- 若获取失败则回退到与 `local_addr` 相同逻辑

### `fn detect_external_ipv4() -> Option<Ipv4Addr>`
- 创建 UDP socket → 连接 `8.8.8.8:80` → 获取本地 IP
- 过滤掉 loopback 和 unspecified

## 测试 (`#[cfg(test)]`)

| 测试 | 验证点 |
|------|--------|
| `studio_local_addr_maps_unspecified_to_loopback` | 未指定 IP 映射为 127.0.0.1 |
| `studio_ext_addr_keeps_loopback_bind_loopback` | 已绑定 loopback 则保持不变 |
| `gateway_bind_candidates_preserves_ephemeral_port_binding` | 临时端口 (0) 只返回单个候选 |
| `gateway_bind_candidates_falls_forward_from_fixed_port` | 固定端口从指定值到 65535 全部候选 |
