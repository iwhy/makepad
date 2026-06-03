# `examples/camera/src/main.rs`

Makepad Camera 示例的入口文件。仅一行：调用 `app.rs` 中定义的 `app_main` 函数启动应用。

```rust
fn main() {
    makepad_example_camera::app::app_main()
}
```

## 逐行解释

| 行 | 说明 |
|---|------|
| 1 | 标准 Rust 程序入口 `fn main()`。 |
| 2 | 调用 `makepad_example_camera::app` 模块的 `app_main()` 宏展开函数。`app_main!(App)` 宏在 `app.rs` 中定义，它会生成启动事件循环的入口逻辑。 |

这个文件非常精简，因为所有应用逻辑（UI 定义、事件处理、摄像头管理）都封装在 `app.rs` 中，`lib.rs` 则负责模块导出和依赖重导出。
