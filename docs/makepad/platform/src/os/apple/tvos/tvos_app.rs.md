# tvos_app.rs — tvOS 应用程序生命周期

**文件路径:** `platform/src/os/apple/tvos/tvos_app.rs`

**核心目的:** 管理 tvOS 应用程序的创建、运行和事件循环。提供 `TvosApp` 结构体，封装 tvOS 平台的 UIApplication 创建、窗口管理和事件分发。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `TvosApp` | 主应用结构体，包含应用状态、窗口引用和事件队列 |
| `TvosAppParams` | 应用参数（继承平台通用 `AppParams`） |

**关键方法:**
- `TvosApp::new_from_app(params, cx, app_ref)` — 创建 tvOS 应用实例，初始化 UIWindow 和 ViewController
- `TvosApp::is_main_thread()` — 检查当前是否为主线程
- `TvosApp::run_app(tick, ...)` — 运行应用主循环：
  - 通过 `CFRunLoopRunInMode` 驱动事件循环
  - 处理 `GCEventViewController` 的远程控制输入
  - 将按键/触摸事件转发至 Makepad 事件系统
  - 管理帧刷新 (CADisplayLink)
- `TvosApp::set_icon()` — tvOS 图标设置（通常无操作）

**实现细节:**
- 使用 `UIApplicationMain` 创建 UIKit 应用环境
- 通过 `GCEventViewController` 实现 Apple TV 遥控器支持
- 使用 `CADisplayLink` 驱动渲染帧
- 窗口管理通过 `_UIWindow_initWithFrame` 和 `_UIWindow_setRootViewController` 实现
- 事件队列将 `TvosEvent` 转换为标准的 `MakepadEvent::KeyDown`/`KeyUp`/`TextInput` 等
- 多点触控通过 UIKit 的 `_UITouch` API 处理
- 使用 `OnceLock` 存储静态回调引用，确保线程安全

**平台集成:** 专用 tvOS (Apple TV)，使用 UIKit 和 GameController 框架
