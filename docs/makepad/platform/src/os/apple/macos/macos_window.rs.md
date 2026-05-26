# macos_window.rs — macOS 窗口管理

**文件路径:** `platform/src/os/apple/macos/macos_window.rs`

**核心目的:** 实现 macOS 平台的窗口创建、管理和事件处理。提供 `MacosWindowHandler`，封装 NSWindow 的完整生命周期，包括创建、配置、绘制和输入处理。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `MacosWindowHandler` | 实现 `PlatformWindowHandler` 的 macOS 窗口处理器 |
| `WindowState` | 窗口状态枚举（`Normal`, `Minimized`, `Maximized`, `Fullscreen`） |

**`MacosWindowHandler` 关键方法:**
- `new_ns_window(...)` — 创建原生 NSWindow：
  - 配置窗口样式（标题栏、缩放、关闭、最小化按钮）
  - 设置 contentView 为 `MetalView`（自定义 MTKView 子类）
  - 注册窗口委托回调
  - 配置接受的文件拖放类型
- `set_window_geom(...)` — 设置窗口位置和大小（使用 `NSWindow_setFrame`）
- `get_window_geom()` — 获取窗口几何信息
- `set_borderless(val)` — 切换无边框模式
- `set_fullscreen(val)` — 切换全屏模式（使用 `toggleFullScreen:`）
- `set_minimum_size(w, h)` — 设置最小窗口尺寸
- `set_title(title)` — 设置窗口标题
- `show_window()` / `hide_window()` — 窗口显隐
- `close_window()` — 关闭窗口，发送 `NSWindow_close`
- `attach_menu(menu)` — 设置窗口菜单栏
- `ns_view()` / `ns_window()` — 获取原生对象引用

**渲染和显示:**
- 使用 `CAMetalLayer` 作为 Metal 渲染目标
- `CVDisplayLink` 驱动显示刷新回调
- `drawRect:` 路由到 Makepad 的渲染流水线
- 视图通过 `NSTrackingArea` 实现鼠标跟踪

**输入处理:**
- 通过 `NSView` 子类拦截 `NSEvent`：
  - `keyDown:` / `keyUp:` — 键盘输入，路由到 `MacosEventConverter`
  - `mouseDown:` / `mouseUp:` / `mouseMoved:` — 鼠标事件
  - `scrollWheel:` — 滚动事件
  - `viewDidEndLiveResize:` — 实时大小调整完成
- 拖放支持：注册 `NSFilenamesPboardType` 文件拖放

**全屏模式:**
- macOS 10.11+ 使用 `toggleFullScreen:` API
- 全屏切换时自动隐藏/显示菜单栏和 Dock
- 窗口状态跟踪（Normal/Minimized/Maximized/Fullscreen）

**平台集成:** macOS 专用，使用 AppKit、Metal、CoreVideo (CVDisplayLink) 和 CoreGraphics
