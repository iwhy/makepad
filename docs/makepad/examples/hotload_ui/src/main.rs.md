# hotload_ui/src/main.rs

演示 Makepad 的 **动态库热加载** 机制。与 counter 示例功能相同，但使用 `makepad_widgets_dll` 替代 `makepad_widgets`。

## 与 counter 的关键区别

- **第1行**：`pub use makepad_widgets_dll as makepad_widgets;` — 通过 dll 形式引入 widget，使得 UI 代码可以在运行时重新编译并热加载
- **第10行**：状态变量命名为 `clicks` 而非 `counter`
- **第42行**：标签文本显示 "Clicks through makepad-widgets-dll: " + state.clicks，标识使用了 dll 模式
- 事件处理和 UI 结构与 counter 完全相同（第63-84行）

## 设计意义

当应用代码作为动态库编译时，Makepad Studio 可以在不重启主进程的情况下重新加载 widget 库，实现即时 UI 更新。这是 Makepad 热重载工作流的典型模式。
