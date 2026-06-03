# text_selection/src/main.rs

简短入口文件，委托给 `makepad_example_text_selection::app::app_main()`。

## 说明

- **第2行**：调用 `app` 模块的 `app_main` 函数，建立 `app_main!(App)` 宏调用
- 这种分层结构允许将示例组织为多文件 crate
