# macos_delegates.rs — macOS 应用和窗口委托

**文件路径:** `platform/src/os/apple/macos/macos_delegates.rs`

**核心目的:** 管理 macOS 应用程序和窗口的 Objective-C 委托回调。实现 `NSApplicationDelegate` 和 `NSWindowDelegate` 协议，将 AppKit 生命周期事件桥接到 Makepad 事件系统。

**关键组件:**

| 类型/函数 | 描述 |
|-----------|------|
| `create_app_delegate(app_state_ptr, activate_cb, deactivate_cb, reopen_cb, file_open_cb)` | 创建 `RinoxAppDelegate`，注册应用生命周期回调 |
| `create_window_delegate(close_cb, zoom_cb, resize_cb)` | 创建窗口委托，注册窗口事件回调 |

**回调类型:**
- 应用级别：
  - `activate_cb`: 应用激活 (`applicationDidBecomeActive`)
  - `deactivate_cb`: 应用失活 (`applicationDidResignActive`)
  - `reopen_cb`: 再次打开（点击 Dock 图标）
  - `file_open_cb`: 文件打开事件 (`application:openFile:`)
- 窗口级别：
  - `close_cb`: 窗口关闭
  - `zoom_cb`: 窗口缩放（绿色按钮）
  - `resize_cb`: 窗口大小改变

**实现细节:**
- 使用 `class!` 宏在运行时注册 Objective-C 子类
- 回调通过 `block!(...)` 宏封装为 Objective-C block
- `app_state_ptr` 通过 `objc_setAssociatedObject` 附加到委托对象
- 窗口关闭检查：`windowShouldClose` 返回 `YES`（允许关闭）
- 支持 macOS 的原生文件打开（通过 `application:openFile:`）

**平台集成:** macOS 专用，使用 AppKit 和 Objective-C 运行时
