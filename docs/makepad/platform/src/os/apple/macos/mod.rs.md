# mod.rs — macOS 平台模块入口

**文件路径:** `platform/src/os/apple/macos/mod.rs`

**核心目的:** macOS 平台的模块声明和公共 API 导出。

**模块声明:**
- `mod macos` — macOS 运行时实现 (`MacosPlatform`)
- `mod macos_app` — macOS 应用生命周期（NSApplication 封装）
- `mod macos_delegates` — NSApplication 和 NSWindow 委托回调
- `mod macos_event` — macOS 原生事件处理（NSEvent）
- `mod macos_stdin` — macOS 标准输入服务
- `mod macos_window` — macOS 窗口管理（NSWindow）

**类型导出:**
- `pub use macos::MacosPlatform;`
- `pub use macos_app::MacosApp;`
- `pub use macos_window::MacosWindowHandler;`
- `pub use macos_event::MacosKeyMapping;`

**平台集成:** 仅在 `cfg(target_os = "macos")` 条件下可用
