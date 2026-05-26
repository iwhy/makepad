# macos_app.rs — macOS 应用程序生命周期

**文件路径:** `platform/src/os/apple/macos/macos_app.rs`

**核心目的:** 管理 macOS 应用程序的完整生命周期。提供 `MacosApp` 结构体，封装 NSApplication 的创建、配置、事件循环和终止流程。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `MacosApp` | macOS 应用主结构体，包含 NSApplication 引用、应用状态和菜单栏配置 |
| `MacosAppParams` | macoS 应用参数（继承通用 `AppParams`） |
| `AppThreadState` | 应用线程状态管理（`Running`, `Terminating`） |

**关键方法:**
- `MacosApp::new()` — 创建 MacosApp 实例：
  - 调用 `NSApplication_sharedApplication` 获取共享应用对象
  - 设置 `activationPolicy` 为 `NSApplicationActivationPolicyRegular`
  - 创建并设置应用委托
  - 构建默认菜单栏（Application 菜单和 File 菜单）
- `MacosApp::finish_launching()` — 完成启动（`NSApp_finishLaunching`）
- `MacosApp::activate_app()` — 激活应用（`NSApp_activateIgnoringOtherApps`）
- `MacosApp::run_app(tick, app_ref)` — 运行应用事件循环：
  - 通过 `NSApplication_run` 进入主运行循环
  - 使用 `CVDisplayLink` 驱动渲染帧
- `MacosApp::terminate()` — 终止应用
- `MacosApp::process_events(tick)` — 处理待处理的事件：
  - 使用 `NSEvent_nextEventMatchingMask` 获取事件
  - 通过 `MacosEventConverter` 转换为 Makepad 事件
  - 分发到活动窗口

**菜单栏管理:**
- `create_default_menu()` — 创建标准 macOS 菜单栏：
  - Application 菜单（关于、偏好设置、服务、隐藏、退出）
  - File 菜单（新建、打开、关闭）
  - Edit 菜单（撤销、重做、剪切、复制、粘贴、全选）
- 使用 `NSMenu` 和 `NSMenuItem` API，通过 `msg_send!` 调用
- 菜单快捷键通过 `keyEquivalent:` 设置
- 菜单动作通过目标-动作模式分发

**事件循环细节:**
- 主循环在 `NSApplication_run` 内部运行
- 事件处理在 `sendEvent:` 重写中进行拦截
- 支持 `NSTimer` 驱动的定时事件
- 通过 `CVDisplayLink` 实现 vsync 同步渲染
- 支持 `PipeOut` 唤醒（通过 `CFRunLoopSource`）

**平台集成:** macOS 专用，使用 AppKit 和 CoreVideo 框架
