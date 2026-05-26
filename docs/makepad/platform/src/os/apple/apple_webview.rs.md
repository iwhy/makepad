# apple_webview.rs — WebView 集成

**文件路径:** `platform/src/os/apple/apple_webview.rs`

**核心目的:** 在 Makepad 应用中集成 WebView 功能。使用 WKWebView（macOS/iOS）提供 HTML 内容渲染、JavaScript 执行和导航控制。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `AppleWebviewWidget` | WebView 小部件实现，支持 `Widget` trait |
| `WebviewNavigator` | WebView 导航控制器（后退、前进、刷新、加载 URL） |
| `WebviewConfig` | WebView 配置选项（JavaScript 启用、内容过滤器等） |

**关键方法:**
- `AppleWebviewWidget::new(cx)` — 创建新的 WebView 实例
- `AppleWebviewWidget::load_url(url)` — 加载指定 URL
- `AppleWebviewWidget::load_html(html, base_url)` — 加载 HTML 字符串
- `AppleWebviewWidget::evaluate_js(js)` — 执行 JavaScript 并返回结果
- `AppleWebviewWidget::go_back()` / `go_forward()` — 导航历史
- `AppleWebviewWidget::reload()` — 刷新当前页面
- `AppleWebviewWidget::set_navigation_delegate(delegate)` — 设置导航事件回调
- `AppleWebviewWidget::take_snapshot()` — 捕获 WebView 截图

**实现细节:**
- 使用 `WKWebView` 作为渲染引擎（macOS 和 iOS 共享 API）
- 通过 `WKUserContentController` 实现 JavaScript ↔ Rust 通信桥
- 导航委托通过 `WKNavigationDelegate` 协议实现：
  - `didStartProvisionalNavigation` — 页面开始加载
  - `didFinishNavigation` — 页面加载完成
  - `didFailNavigation` — 页面加载失败
  - `decidePolicyForNavigationAction` — 导航策略控制
- JavaScript 调用通过 `evaluateJavaScript:completionHandler:` 执行
- 原生事件（鼠标、键盘、滚动）通过 Metal 视图的触控事件传递到 WKWebView
- WebView 尺寸与 Makepad 布局同步

**平台集成:** macOS 和 iOS/tvOS，使用 WebKit 框架
