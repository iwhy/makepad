# browser.rs — Web 浏览器嵌入组件

## 整体职能
`Browser` widget 在 Makepad 应用中嵌入 Web 浏览功能，基于 `webkit2gtk`（Linux）或 `WebView`（Windows/macOS）系统 WebView 引擎，实现对 HTML/CSS/JS 内容的完整渲染支持。

## 主要数据结构
- **`Browser`**：顶层 widget，持有原生 WebView 句柄的抽象 `webview: Option<WebViewHandle>`，以及 `url`（当前 URL）、`navigation_callback`（导航回调）、`loading_state`（加载状态）等字段。
- **`WebViewHandle`**：平台相关的原生 WebView 实例封装。在 Linux 上通过 `webkit2gtk::WebView`，在其他平台通过 WebView2 或 WKWebView。
- **`BrowserAction`**：枚举定义浏览器支持的 Action 类型，包括 `NavigationStarted`、`PageLoaded`、`TitleChanged`、`UrlChanged`、`LoadingStateChanged`。

## 方法与实现逻辑

### `fn script_component` — 脚本注册
注册 `Browser` widget 到脚本运行时。由于该 widget 重度依赖平台原生 API，注册时检查平台支持。

### `fn navigate` — 导航到 URL
接收 URL 字符串。如果是有效的 `http://` / `https://` 或 `file://` URL，调用 `WebViewHandle::load_uri()` 开始加载。同时更新 `self.url` 和 `navigation_callback`。

### `fn draw_walk` — 绘制 WebView 内容
1. 如果尚未创建 WebView 实例且窗口已就绪（有 `Cx` 的原生窗口句柄），在 `cx.begin_native_widget()` 上下文中创建原生 WebView 子窗口。
2. 将 WebView 的视口位置和大小与 widget 的布局矩形同步。
3. 如果 WebView 已创建，调用原生 API 的绘制/合成接口，将 WebView 的内容合成到 Makepad 的绘制帧中。
4. 绘制 `draw_bg` 背景和 `loading_indicator`（加载进度条）。

### `fn handle_event` — 事件处理
- **原生窗口事件**：将 Makepad 的鼠标/键盘事件透传给原生 WebView。
- **WebView 回调**：处理 WebView 触发的 `NavigationStarted`、`PageLoaded`、`TitleChanged` 等信号，转换为 Makepad 的 Action 事件冒泡给父组件。
- **Focus 事件**：当 `Browser` 获得焦点时，将键盘焦点转发给原生 WebView。

### `fn go_back` / `go_forward` — 前进/后退
调用原生 WebView 的历史导航接口：`WebViewHandle::go_back()` / `go_forward()`。更新 `can_go_back` 和 `can_go_forward` 状态供 UI 使用。

### `fn reload` — 重新加载当前页面
调用 `WebViewHandle::reload()`。如果加载失败，显示错误页面。

### `fn execute_javascript` — 执行 JavaScript
通过 `WebViewHandle::evaluate_javascript()` 在页面上下文中执行 JS 代码。支持异步回调，结果通过 `JsResult` action 返回。

### `fn set_user_agent` — 设置 User-Agent
允许外部修改 WebView 的 User-Agent 字符串，用于模拟不同浏览器。

### `fn on_navigation_started` — 导航开始回调
检查即将加载的 URL，可根据 `allowed_domains` 列表决定是否允许导航，或者修改 URL。触发 `BrowserAction::NavigationStarted`。
