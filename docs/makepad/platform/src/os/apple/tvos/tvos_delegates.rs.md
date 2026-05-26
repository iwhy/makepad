# tvos_delegates.rs — tvOS 应用委托回调

**文件路径:** `platform/src/os/apple/tvos/tvos_delegates.rs`

**核心目的:** 管理 tvOS 应用程序的 Objective-C 委托回调，桥接 UIKit 应用程序生命周期事件和远程控制输入到 Makepad 事件系统。

**关键组件:**

| 类型/函数 | 描述 |
|-----------|------|
| `AppDelegateCallbacks` | 结构体，保存应用启动/激活/停用/终止等回调函数的引用 |
| `create_tvos_app_delegate(app_cb, activate_cb, deactivate_cb, terminate_cb)` | 创建并注册 `TvosAppDelegate`，连接 UIKit 生命周期事件。返回 `Id<UIApplicationDelegate>` |
| `set_remote_control_callbacks(playpause_cb, menu_cb, select_cb, swipe_cb, siri_cb)` | 为 GCEventViewController 注册远程控制按键和手势回调 |

**实现细节:**
- 使用 Objective-C 消息发送 (`msg_send!`) 调用 UIKit API
- 使用 `block!` 宏从 Rust 闭包创建 Objective-C blocks
- 通过 `GCController` 和 `GCEventViewController` 处理 Siri Remote 输入
- 将 Apple TV 的 swipe 手势映射到内部 `SwipeDirection` 枚举
- 应用生命周期通过 `_UIApplicationDelegate` protocol 方法的 block 回调桥接

**平台集成:** tvOS 专用，直接操作 UIKit 和 GameController 框架
