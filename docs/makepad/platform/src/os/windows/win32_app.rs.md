# File: win32_app.rs

- **核心用途**: 实现 Makepad 应用在 Windows 平台上的生命周期管理，包括 Win32 消息循环（PeekMessage/DispatchMessage）、应用启动/退出流程、计时器管理和跨线程事件调度。
- **所属层级**: 平台层 · 应用生命周期

---

## 类型与结构体

| 名称 | 可见性 | 简介 |
|------|--------|------|
| `Win32App` | `pub` | Win32 应用主结构体，管理消息循环、事件队列、窗口注册类、计时器和运行状态。 |
| `AppMode` | `pub` | 应用模式枚举：`Windowed`（普通窗口应用）、`Xaml`（UWP XAML 宿主应用）。 |
| `TimerEntry` | `pub` | 计时器条目，包含周期和回调信息。 |

## 核心方法

| 方法 | 可见性 | 简述 |
|------|--------|------|
| `new(app_mode) -> Self` | `pub` | 创建 Win32App 实例，注册默认窗口类（`WNDCLASSW`），初始化事件队列。 |
| `run(main_window) -> Result<()>` | `pub` | 启动应用主循环：持续 PeekMessage → TranslateMessage → DispatchMessage，空闲时调用 `update` 和 `draw`。 |
| `quit(exit_code)` | `pub` | 发送 WM_QUIT 消息终止消息循环。 |
| `set_timer(id, interval, callback)` | `pub` | 创建或更新计时器（通过 `SetTimer` + 回调映射）。 |
| `kill_timer(id)` | `pub` | 移除指定计时器。 |
| `post_event(event)` | `pub` | 跨线程投递事件到主消息循环（通过 `PostThreadMessage` 或自定义队列）。 |
| `process_messages()` | `pub` | 非阻塞处理所有待处理消息（单次消息泵）。 |
| `hwnd(&self) -> HWND` | `pub` | 返回主窗口句柄。 |
| `window_class(&self) -> &WNDCLASSW` | `pub` | 返回注册的窗口类信息。 |

## 实现细节

- 消息循环使用 `PeekMessageW`（非阻塞）而非 `GetMessageW`（阻塞），以允许 Makepad 的异步渲染和事件处理在空闲时继续进行。
- `run` 方法包含 `MSG` 结构体，每帧持续泵送消息直到收到 `WM_QUIT`。
- 窗口类注册使用 `RegisterClassW`，支持自定义 `hbrBackground`、`hCursor`、`hIcon`。
- 计时器回调通过 `WM_TIMER` 的 `wParam` 携带计时器 ID，匹配到注册的 `TimerEntry`。
- `post_event` 使用自定义的线程安全 `Receiver<Event>` 通道，在主循环中被 `PeekMessage` 间隙检查。

## 平台集成

- 是 Win32 窗口应用在 Makepad 平台抽象层的入口点。
- 负责将 Win32 窗口消息（`WM_SIZE`、`WM_PAINT`、`WM_CLOSE` 等）转化为 Makepad 内部事件。
- 通过 `SetWindowLongPtrW(GWL_USERDATA)` 将 `Win32Window` Rust 对象指针绑定到窗口实例。
- 支持 MFC/ATL 样式的 `_Module` 初始化（`CoInitializeEx`、` OleInitialize` 等）的调用。
