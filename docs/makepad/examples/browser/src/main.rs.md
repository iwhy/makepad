# browser/src/main.rs

演示 CEF（Chromium Embedded Framework）和原生浏览器组件的嵌入，包含 Tab 布局和 PortalList 中嵌入浏览器 widget 的高级用法。

## 整体结构

### UI 定义（第25-161行）

- **第29-78行**：`NativeBrowserList` 注册 — 包装 PortalList，每个条目包含一个 `Browser` widget，配置 `BrowserBackend.Native`
- **第80-146行**：`AppDock` — 使用 Dock/DockTabs 双标签布局：
  - **CEF Tab**（第106-112行）：全屏 CEF Chromium 浏览器，打开 google.nl
  - **Native Tab**（第114-145行）：包含 `NativeBrowserList`，在 PortalList 中嵌入 20 个原生浏览器实例

### NativeBrowserList widget（第169-235行）

关键实现：
- `active` 标志控制浏览器可见性（Tab 切换时管理资源）
- `bindings: HashMap<WidgetUid, usize>` — 跟踪 widget → 数据绑定
- `browsers: HashMap<WidgetUid, BrowserRef>` — 持久化浏览器引用
- `set_active_internal`：当标签切换时，控制所有浏览器实例的可见性
- `bind_item`：绑定 PortalList 条目到浏览器，设置标题、URL，复用现有浏览器实例
- `draw_walk`：在 PortalList 虚拟列表迭代中调用 `bind_item` 和 `draw_all_unscoped`

### 事件处理（第248-273行）

- `handle_actions`：监听 `DockAction::TabWasPressed` 事件，切换 Native Tab 时调用 `set_native_tab_active`
- `handle_event`：`Startup` 事件时禁用 Native Tab（避免启动时创建 20 个浏览器实例）

### 主入口（第276-368行）

`main` 函数 / `app_main` 函数处理 CEF 引导流程：
- `makepad_cef::reexec_into_app_bundle_if_needed()` — macOS 应用包重新执行
- `makepad_cef::bootstrap()` — CEF 引导
- `makepad_cef::initialize()` — CEF 初始化
- 手动构造 `Cx` 和事件循环（不使用 `app_main!` 宏的自动模式）
- 热重载支持（`LiveEdit` 事件处理）

## 关键 API

- `Browser` widget — 浏览器嵌入式组件（`BrowserBackend::Native` / `BrowserBackend::CEF`）
- `BrowserRef::set_url()` / `set_visible()`
- `DockAction::TabWasPressed` — Dock 标签切换事件
- `makepad_cef` — CEF 集成库
