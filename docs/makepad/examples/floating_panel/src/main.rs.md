# floating_panel/src/main.rs

演示 macOS 平台上浮动面板窗口（NSPanel）的创建和交互。

## 整体结构

- **第7-31行**：`script_mod!` UI 模板定义 — `Card` 和 `Tag` 两个可复用组件
- **第33-227行**：双窗口 UI 定义：
  - **主窗口**（第36-134行）：显示说明信息、状态卡片、配置卡片
  - **panel_window**（第136-226行）：浮动面板窗口，`show_caption_bar: false` 隐藏原生标题栏，包含文本输入框、按钮和拖拽目标区域
- **第230-242行**：App 结构体包含面板配置状态、窗口 ID、交互计数和拖拽原点
- **第244-270行**：`configure_panel` 方法配置 macOS 浮动面板特性；`set_status` 在双窗口间同步状态
- **第272-298行**：`MatchEvent::handle_actions` — 处理面板按钮点击、输入框回车和文本变更事件
- **第301-357行**：`AppMain::handle_event` — 处理：
  1. `Event::Startup`：初始化面板
  2. `Event::WindowDragQuery`：检测鼠标在面板标题区域的拖拽，返回 `Caption` 响应实现自定义拖拽
  3. `drag_target` 的拖拽检测（MouseDown/MouseMove/MouseUp）

## 关键 API

- `MacosWindowConfig::floating_panel()` — macOS 浮动面板配置
- `MacosWindowChrome::Borderless` — 无边框窗口
- `WindowDragQuery` / `WindowDragQueryResponse` — 自定义窗口拖拽
- `window.show_caption_bar` — 隐藏原生标题栏
