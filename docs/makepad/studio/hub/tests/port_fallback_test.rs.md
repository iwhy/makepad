# port_fallback_test.rs

## 概述

测试 `StudioHub::start_headless` 在请求端口被占用时的端口回退行为。确保 headless 后端能自动绑定到比请求端口更高的可用端口，并且回退后的 HTTP 健康检查端点正常响应。

## 测试场景

### `headless_backend_falls_back_to_higher_port_when_requested_port_is_busy`

- **场景**: 请求端口已被 `TcpListener` 占用。
- **流程**:
  1. 通过 `find_occupied_port_below_max()` 获取一个已被占用的端口。
  2. 配置 `HubConfig` 的 `listen_address` 指向该端口，`enable_in_process_gateway: false`。
  3. 调用 `StudioHub::start_headless(config)` 启动后端。
  4. 验证 `backend.listen_address.port()` 大于 `busy_port`（自动回退到更高端口）。
  5. 通过 `TcpStream` 发送 HTTP `GET /$studio_health` 请求。
  6. 验证响应以 `HTTP/1.1 200 OK` 开头。

## 测试模式

- **端口回退**: headless 模式下如果端口被占用，自动尝试更高端口。
- **HTTP 健康检查**: 回退后的端口仍然提供 `/$studio_health` 端点。
- **绑定检查**: `find_occupied_port_below_max()` 辅助函数创建一个临时 listener 来占位端口。
