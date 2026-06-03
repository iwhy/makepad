# `lib.rs` — Studio Hub Backend 模块声明

## 文件位置
- 路径: `studio/hub/src/lib.rs`
- 行数: 20 行
- 作用: 声明 hub backend 的所有公有/私有模块，并重导出关键类型到 crate 根

## 模块声明

```rust
pub mod ai_manager;
pub mod build_manager;
pub mod dispatch;
pub mod gateway;
pub mod hub;
pub mod log_store;
pub mod script_manager;
pub mod terminal_manager;
pub mod test_support;  // #[doc(hidden)]
pub mod virtual_fs;
mod worker_pool;       // 私有模块
```

11 个模块，其中 `worker_pool` 是私有的（仅 crate 内部可见），`test_support` 标记为 `#[doc(hidden)]`。

## 重导出 (`pub use`)

| 目标类型 | 来源模块 | 说明 |
|---------|---------|------|
| `AiManager` | ai_manager | AI Agent 管理器 |
| `BuildManager` | build_manager | 构建进程管理器 |
| `HubCore`, `HubEvent` | dispatch | 核心事件调度器 |
| `HubConfig`, `HubConnection`, `HubHandle`, `MountConfig`, `StudioHub` | hub | Hub 入口和连接 |
| `LogQuery`, `LogStore` | log_store | 日志存储和查询 |
| `ProfilerQuery`, `ProfilerStore` | log_store | 性能分析数据存储 |
| `ScriptId`, `ScriptManager`, `MAKEPAD_SPLASH_RUNNABLE` | script_manager | 脚本运行管理器 |
| `VirtualFs` | virtual_fs | 虚拟文件系统 |

## 设计观察

- `worker_pool` 不重导出，表明它是纯内部实现细节
- `test_support` 标记 `#[doc(hidden)]`，但仍是 `pub mod`，测试代码可通过 `crate::test_support::tempdir()` 访问
- `GatewayHandle` 类型也未重导出，网关仅通过 hub 模块间接暴露
