# apple_classes.rs — 运行时 Objective-C 类注册

**文件路径:** `platform/src/os/apple/apple_classes.rs`

**核心目的:** 在运行时注册和管理 Makepad 使用的自定义 Objective-C 类。使用 `OnceLock` 确保线程安全的惰性初始化，避免类重复注册。

**关键函数:**

| 函数 | 描述 |
|------|------|
| `get_view_class()` | 获取/注册 `RinoxView` 类 — NSView/UIView 的子类，用于自定义绘制和事件处理 |
| `get_view_controller_class()` | 获取/注册 `RinoxViewController` 类 — UIViewController 的子类，用于视图控制器管理 |
| `get_app_delegate_class()` | 获取/注册 `RinoxAppDelegate` 类 — UIApplicationDelegate 的子类，用于应用生命周期 |
| `get_window_delegate_class()` | 获取/注册 `RinoxWindowDelegate` 类 — NSWindowDelegate 的子类（macOS） |
| `get_metal_view_class()` | 获取/注册 `MetalView` 类 — MTKView 的子类，用于 Metal 渲染（macOS） |
| `class_names()` | 返回所有已注册类名的字符串表示 |

**实现细节:**
- 每个类使用独立的 `OnceLock<ClassState>` 实现惰性初始化
- 使用 `objc_allocateClassPair` / `objc_registerClassPair` API 动态注册类
- 类方法通过 `class_addMethod` 添加，使用 `extern "C"` 函数指针
- 实例变量通过 `class_addIvar` 添加（用于存储回调指针）
- 采用工厂模式：相同类名只注册一次，后续调用返回缓存引用
- 基类选择：macOS 使用 `NSView`, iOS/tvOS 使用 `UIView`

**平台集成:** macOS、iOS、tvOS 共享此模块，基类选择通过条件编译处理
