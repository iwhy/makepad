# File: win32_window.rs

- **核心用途**: 实现 Win32 窗口的完整生命周期管理，包括窗口创建、消息处理（WindowProc）、事件映射（WM_* → Makepad Event）、缩放/DPI 处理、鼠标/键盘/触控输入处理、窗口移动/调整大小、菜单/标题栏控制以及无障碍支持。
- **所属层级**: 平台层 · 窗口管理
- **涉及行数**: ~5600 行（该平台第二大的文件）

---

## 类型与结构体

| 名称 | 可见性 | 简介 |
|------|--------|------|
| `Win32Window` | `pub` | Win32 窗口主结构体，管理窗口状态、输入状态、DPI 缩放、事件回调映射、无障碍代理。 |
| `WindowEvent` | `pub` | Windows 特定窗口事件封装，用于将 Win32 消息转换为 Makepad 事件。 |
| `Modifiers` | `pub` | 键盘修饰键状态（Ctrl, Alt, Shift, Win）。 |
| `CursorKind` | `pub` | 光标类型枚举，映射到 Win32 标准光标 IDC_*。 |
| `CursorState` | `pub` | 光标状态：`Visible`, `Hidden`, `Confined`。 |
| `WindowClass` | `pub` | 窗口类描述，包含类名、图标、光标、背景画刷。 |

## 核心方法

### 窗口创建与生命周期

| 方法 | 可见性 | 简述 |
|------|--------|------|
| `new(class) -> Self` | `pub` | 创建窗口实例，注册窗口类，创建窗口句柄。 |
| `show(cmd_show)` | `pub` | 显示窗口（`ShowWindow` + `UpdateWindow`）。 |
| `close()` | `pub` | 关闭窗口（发送 `WM_CLOSE`）。 |
| `destroy()` | `pub` | 销毁窗口。 |
| `hwnd(&self) -> HWND` | `pub` | 返回窗口句柄。 |
| `is_minimized() -> bool` | `pub` | 检查窗口是否最小化。 |
| `is_maximized() -> bool` | `pub` | 检查窗口是否最大化。 |

### 窗口消息处理（WindowProc）

| 方法 | 可见性 | 简述 |
|------|--------|------|
| `window_proc(hwnd, msg, wparam, lparam) -> LRESULT` | `pub` | 主窗口过程，通过 `match msg` 分发所有 `WM_*` 消息到对应的处理器。 |
| `def_window_proc(hwnd, msg, wparam, lparam) -> LRESULT` | `pub` | 默认窗口过程调用（`DefWindowProcW`）。 |

### 事件映射（WM_* → Makepad）

Win32Window 的 WndProc 处理以下关键消息：

| 消息 | 处理摘要 |
|------|----------|
| `WM_PAINT` | 触发布局计算和重绘请求 |
| `WM_SIZE` | 更新窗口尺寸缓存，发送 Resize 事件 |
| `WM_DPICHANGED` | 更新 DPI 缩放因子，调整窗口大小以适应新 DPI |
| `WM_MOUSEMOVE` | 更新鼠标位置，发送 CursorMove 事件 |
| `WM_LBUTTONDOWN/UP` | 鼠标左键按下/释放，发送 MouseDown/MouseUp |
| `WM_RBUTTONDOWN/UP` | 鼠标右键按下/释放，发送 ContextMenu 请求 |
| `WM_MOUSEWHEEL` | 鼠标滚轮事件，转换为 Makepad Scroll 事件 |
| `WM_KEYDOWN/UP` | 键盘按键按下/释放，映射到 Makepad KeyCode |
| `WM_CHAR` | 文本输入，发送 TextInput 事件 |
| `WM_SYSKEYDOWN` | 系统按键（Alt+组合键）处理 |
| `WM_SETCURSOR` | 光标设置，返回窗口光标或 HTCLIENT 区域默认光标 |
| `WM_NCHITTEST` | 非客户区命中检测，支持自定义标题栏 |
| `WM_NCCALCSIZE` | 非客户区计算，用于隐藏系统标题栏 |
| `WM_ENTERSIZEMOVE / WM_EXITSIZEMOVE` | 窗口拖拽生命周期 |
| `WM_CLOSE` | 发送 Close 事件到应用 |
| `WM_DESTROY` | 清理窗口资源，发送 Destroy 事件 |
| `WM_TOUCH` | 触控输入处理，通过 `GetTouchInputInfo` |
| `WM_ACTIVATE` | 窗口激活状态变化 |
| `WM_SETFOCUS / WM_KILLFOCUS` | 焦点事件 |
| `WM_MENUCOMMAND` / `WM_COMMAND` | 菜单/控件命令 |

### 窗口控制

| 方法 | 可见性 | 简述 |
|------|--------|------|
| `set_title(title)` | `pub` | 设置窗口标题（`SetWindowTextW`）。 |
| `set_position(x, y, w, h)` | `pub` | 设置窗口位置和尺寸。 |
| `set_min_size(w, h)` | `pub` | 设置最小窗口尺寸。 |
| `set_max_size(w, h)` | `pub` | 设置最大窗口尺寸。 |
| `set_cursor(kind)` | `pub` | 设置窗口光标类型（`SetCursor` + 缓存）。 |
| `set_cursor_state(state)` | `pub` | 设置光标可见性、裁剪等状态。 |
| `set_fullscreen(fullscreen)` | `pub` | 切换全屏模式（`hwnd_control::toggle_fullscreen`）。 |
| `set_resizable(resizable)` | `pub` | 切换窗口是否可调整大小（更新 `WS_THICKFRAME` 样式）。 |
| `set_maximizable(maximizable)` | `pub` | 切换窗口是否可最大化。 |
| `center()` | `pub` | 将窗口居中到屏幕工作区。 |
| `request_redraw()` | `pub` | 请求窗口重绘（`InvalidateRect` + `UpdateWindow`）。 |

### 输入/绘图状态

| 方法 | 可见性 | 简述 |
|------|--------|------|
| `begin_paint() -> PAINTSTRUCT` | `pub` | 开始绘制（`BeginPaint`），返回绘图区域信息。 |
| `end_paint(ps)` | `pub` | 结束绘制（`EndPaint`）。 |
| `get_client_rect() -> (i32, i32, i32, i32)` | `pub` | 获取客户区矩形。 |
| `get_window_rect() -> (i32, i32, i32, i32)` | `pub` | 获取窗口矩形。 |
| `dpi_scale() -> f64` | `pub` | 返回当前 DPI 缩放因子。 |
| `set_event_handler(handler)` | `pub` | 设置窗口事件回调。 |

### 无障碍（UI Automation）

| 方法 | 可见性 | 简述 |
|------|--------|------|
| `create_uia_provider()` | `pub` | 创建 UI Automation 提供者。 |
| `uia_get_pattern_provider(pattern_id) -> IUnknown` | `pub` | 返回指定 UIA 模式提供者。 |
| `uia_get_property_value(property_id) -> Variant` | `pub` | 返回指定 UIA 属性的值。 |

## 实现细节

- `window_proc` 使用 `match msg` + `WM_*` 常量模式匹配分发，每个消息在独立的处理分支中执行。
- 窗口实例指针通过 `SetWindowLongPtrW(GWL_USERDATA)` 存储，WndProc 入口处通过 `GetWindowLongPtrW` 获取 Rust `&mut Win32Window` 引用。
- DPI 感知通过 `SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2)` 启用每监视器 DPI 感知。
- `WM_NCCALCSIZE` 处理中返回 `0`（WVR_HREDRAW | WVR_VREDRAW）以消除标准标题栏，实现自定义标题栏。
- `WM_NCHITTEST` 根据预定义的掩码区域返回 `HTCLIENT` / `HTCAPTION` / `HTCLOSE` / `HTMINBUTTON` / `HTMAXBUTTON` 等，实现窗口控件的自定义点击区域。
- 触控（`WM_TOUCH`）支持多点触控手势，通过 `RegisterTouchWindow` 启用。
- 键盘 KeyCode 映射从 `wParam` 的虚拟键码通过查找表或 `MapVirtualKeyW` 转换为 Makepad 的 `KeyCode` 枚举。

## 平台集成

- 纯 Win32 API 实现，使用 `windows-sys` crate 原生调用。
- `cursor.rs` 管理 `IDC_ARROW`、`IDC_HAND`、`IDC_IBEAM`、`IDC_CROSS`、`IDC_SIZEALL` 等标准光标加载。
- 无障碍集成通过 UIAutomation `IRawElementProviderSimple` 接口实现屏幕阅读器支持。
- 窗口类注册在 `Win32App::new` 阶段完成，`Win32Window::new` 仅创建窗口实例。
- 与 `hwnd_control.rs` 配合管理窗口样式更改（如全屏切换）。
