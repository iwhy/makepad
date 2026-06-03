# main.rs

应用程序入口文件。

```rust
fn main() {
    makepad_example_uizoo::app::app_main()
}
```

- `fn main()`: Rust 标准入口点。
- 直接委托给 `makepad_example_uizoo::app` 模块中的 `app_main()` 函数。
- `app_main!` 宏在 `app.rs` 中展开，会生成实际的 `main()` 入口逻辑并处理平台初始化（窗口、事件循环等）。
- 这一层间接引用的存在是因为 Makepad 需要捕获平台相关的启动逻辑。
