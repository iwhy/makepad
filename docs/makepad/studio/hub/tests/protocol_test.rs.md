# protocol_test.rs

## 概述

测试 Makepad Studio Hub 协议层的序列化/反序列化正确性。覆盖 `ClientToHubEnvelope`、`HubToClient` 和 `QueryId` 的二进制与 JSON 格式的 roundtrip。

## 测试场景

### `query_id_layout_roundtrip`

- **场景**: 验证 `QueryId` 的结构布局：`QueryId::new(client_id, counter)` 能正确提取 `client_id`（按 `QUERY_ID_CLIENT_LANES` 取模）和 `counter`。
- **流程**:
  1. 创建 `ClientId(42)` 和 `QueryId::new(client, 123456789)`。
  2. 断言 `qid.client_id()` 返回 `ClientId(42 % QUERY_ID_CLIENT_LANES)`。
  3. 断言 `qid.counter()` 返回 `123456789`。

### `ui_envelope_binary_and_json_roundtrip`

- **场景**: 验证 `ClientToHubEnvelope` 的二进制（`serde_bin`）和 JSON 序列化/反序列化一致性。
- **流程**:
  1. 构造一个 `ClientToHub::LoadFileTree` 消息的 envelope。
  2. 序列化为二进制 → 反序列化 → 断言 `query_id` 匹配。
  3. 序列化为 JSON → 反序列化 → 断言 `query_id` 匹配。

### `studio_to_ui_binary_and_json_roundtrip`

- **场景**: 验证 `HubToClient` 枚举的二进制和 JSON 序列化/反序列化一致性。
- **流程**:
  1. 构造 `HubToClient::Hello { client_id: ClientId(7) }`。
  2. 二进制序列化 → 反序列化 → 匹配 `Hello` 变体 → 断言 client_id 正确。
  3. JSON 序列化 → 反序列化 → 同样断言。

## 测试模式

- **协议层单元测试**：不涉及网络或后端启动。
- **双重序列化验证**：同时测试二进制（`SerBin`/`DeBin`）和 JSON（`SerJson`/`DeJson`）格式。
- **Roundtrip 一致性**：序列化后反序列化应得到与原始值相等的对象。
