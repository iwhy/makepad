# macos.rs — macOS 平台实现

**文件路径:** `platform/src/os/apple/macos/macos.rs`

**核心目的:** 实现 `MacosPlatform` 结构体，作为 Makepad 框架中 macOS 平台抽象的核心。提供窗口管理、应用生命周期、事件循环、显示器和剪贴板操作。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `MacosPlatform` | 实现 `PlatformTrait`，macOS 平台的主要抽象 |
| `MacosWindowHandler` | 实现 `PlatformWindowHandler`，管理单个 NSWindow |
| `MacosMonitorInfo` | 显示器信息结构体 |

**`MacosPlatform` 关键方法:**
- `new(cx)` — 初始化 macOS 平台：
  - 初始化共享 NSApplication，配置激活策略
  - 创建应用委托，注册生命周期回调
  - 设置默认菜单栏（Apple、File、Edit、Window、Help 菜单）
  - 同步 CVDisplayLink 状态
- `get_date()` — 获取当前日期时间
- `get_screen_size()` — 获取主屏幕尺寸
- `get_main_thread_id()` — 获取主线程 ID
- `run_app(tick, app_ref)` — 进入 NSApplication 主运行循环：
  - 使用 `CVDisplayLink` 驱动渲染
  - 支持 `PipeOut` 信号唤醒
  - 通过 `NSEvent` 队列驱动事件处理
- `create_window(...)` — 创建新的 NSWindow：
  - 配置窗口样式、初始位置和大小
  - 创建 MetalView 作为内容视图
  - 设置窗口委托
  - 注册接受的文件类型
- `get_window_count()` — 获取窗口数量
- `get_all_monitors()` — 获取所有显示器信息（分辨率、位置、主显示器标志）
- `get_drag_and_drop_state()`/`set_drag_and_drop_state()` — 拖放状态管理

**剪贴板操作:**
- `get_clipboard()` — 读取系统剪贴板（通过 `NSPasteboard_generalPasteboard`）
- `set_clipboard(s)` — 写入系统剪贴板

**显示器枚举:**
- 使用 `NSScreen_screens` 获取所有屏幕
- 每个屏幕的信息包括：分辨率、DPI、位置（相对于虚拟空间原点）
- 主显示器通过 `screens[0]` 确定

**光标管理:**
- `show_cursor(show)` — 显示/隐藏光标（通过 `NSCursor`）
- `set_cursor_style(style)` — 设置光标样式（箭头、I-beam、十字、手型等）

**休眠管理:**
- `set_sleep(sleep)` — 使用 `NSProcessInfo` 的 `beginActivityWithOptions`/`endActivity` 防止系统休眠

**菜单栏:**
- 完整的 macOS 菜单栏包括：Apple 菜单、File、Edit、View、Window、Help
- 支持动态菜单更新
- 服务菜单集成

**平台集成:** macOS 专用，使用 AppKit、CoreVideo (CVDisplayLink)、Metal、CoreGraphics 和 CoreText
