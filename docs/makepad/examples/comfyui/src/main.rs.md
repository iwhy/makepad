# `examples/comfyui/src/main.rs`

ComfyUI 示例的入口文件。

```rust
fn main() {
    makepad_example_comfyui::app::app_main()
}
```

## 逐行解释

| 行 | 说明 |
|---|------|
| 1 | 标准程序入口。 |
| 2 | 调用 `app.rs` 中 `app_main!(App)` 宏生成的启动函数。 |

示例通过 `lib.rs` 导出 `makepad_widgets` 依赖，并声明 `app` 和 `edmx` 两个模块。
