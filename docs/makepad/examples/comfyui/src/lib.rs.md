# `examples/comfyui/src/lib.rs`

ComfyUI 示例的库根文件。

```rust
pub use makepad_widgets;
pub mod app;
pub mod edmx;
```

## 逐行解释

| 行 | 说明 |
|---|------|
| 1 | 重导出 `makepad_widgets` crate，使 `app_main!` 等宏可以找到该依赖。 |
| 2 | 声明 `app` 模块——主 UI 逻辑和 LLM + ComfyUI + EMDX 工作流。 |
| 3 | 声明 `edmx` 模块——三星 EMDX 电子纸显示器的 UDP Wake-on-LAN 和 TCP/MDC 通信协议实现。 |

这里将网络通信协议（edmx）与 UI 和业务逻辑（app）分离，是良好的模块化设计。
