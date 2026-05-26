# apple_util.rs — Apple 平台通用工具

**文件路径:** `platform/src/os/apple/apple_util.rs`

**核心目的:** 提供 Apple 平台共用的实用函数和扩展。涵盖光标加载、NSString/CFString 转换、事件通知、URL 处理、系统声音和菜单构建等功能。

**光标管理:**
- `load_cursor_from_tiff(name)` — 从 TIFF 数据加载 macOS 光标
- `CursorKind` 枚举 — 定义支持的系统光标类型（箭头、I-beam、十字、手型、水平/垂直调整等）
- macOS 使用 `NSCursor` 系统光标；iOS/tvOS 光标操作为空操作

**字符串转换:**
- `nsstring_from_str(s)` / `str_from_nsstring(s)` — Rust `&str` ↔ `Id<NSString>` 双向转换
- `cfstring_from_str(s)` / `str_from_cfstring(s)` — Rust `&str` ↔ `CFStringRef` 双向转换
- 使用 CoreFoundation 的 `CFStringGetCString`/`CFStringCreateWithCString` 或 Objective-C 的 `NSString` 方法

**事件通知:**
- `post_notification(name)`, `post_notification_with_object(name, obj)` — 发布 macOS 分布式通知 (`NSDistributedNotificationCenter`)
- `post_local_notification(name)` — 发布本地通知 (`NSNotificationCenter`)

**URL 处理:**
- `open_url(url)` — 使用 `NSWorkspace_openURL` 在默认浏览器中打开 URL

**系统声音:**
- `play_system_beep()` — 播放系统提示音（通过 `NSBeep()`）

**菜单构建:**
- `MenuAction` 枚举 — 定义菜单动作类型
- `create_menu(title, items)` — 构建 NSMenu 对象
- `add_menu_item(menu, title, action, key_equiv)` — 向菜单添加带快捷键的项目
- 使用 `NSMenu` 和 `NSMenuItem` API 通过 `msg_send!` 操作

**其他工具:**
- `app_kit_main_thread_id()` — 获取 AppKit 主线程 ID（macOS）
- `dispatch_sync_main(f)` — 在主线程同步执行 block
- `dispatch_async_main(f)` — 在主线程异步执行 block

**平台集成:** macOS、iOS、tvOS 条件编译；部分功能（光标、NSWorkspace）仅限 macOS
