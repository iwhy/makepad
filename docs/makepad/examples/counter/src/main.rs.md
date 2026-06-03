# counter/src/main.rs

这是一个简单的计数器示例，演示 Makepad 使用 `script_mod!` DSL 和 `script_eval!` 进行状态管理的基础模式。

## 整体结构

- **第1行**：`pub use makepad_widgets;` — 导出 widget crate，供测试使用
- **第3行**：引入 widget 库
- **第5行**：`app_main!(App);` — 声明 App 为入口结构体
- **第7-41行**：`script_mod!` — UI 定义与状态声明。使用 `let state = { counter: 0 }` 声明脚本层状态，通过 `mod.state = state` 将其注册到脚本模块中
- **第43-47行**：App 结构体包含 `#[live] ui: WidgetRef`，用于在 Rust 侧访问脚本定义好的 UI
- **第49-58行**：`MatchEvent::handle_actions` — 监听 `increment_button` 的点击事件。当按钮被点击时，通过 `script_eval!` 宏在脚本层执行 `mod.state.counter += 1` 更新计数，然后调用 `ui.main_view.render()` 触发界面重绘
- **第60-70行**：`AppMain` trait 实现 — `script_mod` 负责注册基础 widgets 并加载自己的 script_mod；`handle_event` 将事件分发给 `MatchEvent` 和 widget 树

## 关键 API 模式

- `script_eval!(cx, { ... })` — 在 Rust 中执行脚本层代码，可以直接读写 `mod.state` 等脚本变量
- `on_render: ||{ ... }` — 脚本层回调，每当视图需要重绘时执行，在回调中动态创建 Label 并读取 `state.counter`
- `self.ui.button(cx, ids!(increment_button)).clicked(actions)` — 通过 `WidgetRef` 查找组件并检测事件
