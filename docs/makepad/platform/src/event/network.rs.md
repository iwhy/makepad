# `network.rs` — 网络响应事件

## 概述

该文件是网络事件模块的极简入口，仅包含两行类型重导出定义。

```rust
pub type NetworkResponse = crate::makepad_network::NetworkResponse;
pub type NetworkResponsesEvent = Vec<NetworkResponse>;
```

### `NetworkResponse`
类型别名，指向 `makepad_network` 模块中的 `NetworkResponse` 类型。

### `NetworkResponsesEvent`
`Vec<NetworkResponse>` 的类型别名。一次网络事件可能携带多个响应（如批量请求完成）。此类型被 `Event::NetworkResponses(NetworkResponsesEvent)` 变体使用。
