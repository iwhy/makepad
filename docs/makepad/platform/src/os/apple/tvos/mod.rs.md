# mod.rs — tvOS 平台模块入口

**文件路径:** `platform/src/os/apple/tvos/mod.rs`

**核心目的:** tvOS 平台的模块声明和公共 API 导出。作为 tvOS 特定实现的根模块。

**模块声明:**
- `mod tvos` — tvOS 运行时实现
- `mod tvos_app` — tvOS 应用程序生命周期管理
- `mod tvos_delegates` — tvOS 应用代理回调处理
- `mod tvos_event` — tvOS 事件处理

**类型:** `pub use tvos::TvosPlatform;` — 公开 tvOS 平台实现

**平台集成:** 仅在 `cfg(target_os = "tvos")` 条件下可用
