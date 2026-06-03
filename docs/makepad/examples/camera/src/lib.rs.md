# `examples/camera/src/lib.rs`

Camera 示例的库根文件，负责模块声明和依赖重导出。

```rust
pub use makepad_widgets;
pub mod app;
```

## 逐行解释

| 行 | 说明 |
|---|------|
| 1 | 重导出 `makepad_widgets` crate，使下游可以方便地引用。这是 Makepad 应用的标准模式，因为 `app_main!` 宏需要 widget 框架支持。 |
| 2 | 声明 `app` 子模块。所有 UI 逻辑和摄像头控制逻辑都位于 `app.rs` 中。 |

这种 `lib.rs` 结构是所有 Makepad 示例应用的标准入口模式：重导出 widget 框架 + 暴露 `app` 模块。
