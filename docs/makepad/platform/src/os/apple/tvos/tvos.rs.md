# tvos.rs — tvOS 平台实现

**文件路径:** `platform/src/os/apple/tvos/tvos.rs`

**核心目的:** 实现 `TvosPlatform` 结构体，作为 Makepad 框架中 tvOS 平台抽象的核心。提供窗口创建、应用生命周期、事件循环和平台查询功能。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `TvosPlatform` | 实现 `PlatformTrait`，是 tvOS 平台的主要抽象 |
| `TvosWindowHandler` | 实现 `PlatformWindowHandler`，管理单个窗口 |
| `TvosDisplayLink` | CADisplayLink 的包装，驱动渲染帧同步 |
| `TvosAppParams` | tvOS 特定的应用参数 |

**`TvosPlatform` 关键方法:**
- `new()` — 初始化 tvOS 平台，设置基本 App State
- `get_date()` — 获取当前日期时间
- `get_screen_size()` — 获取屏幕尺寸（通过 UIScreen）
- `get_main_thread_id()` — 获取主线程 ID
- `run_app(tick, app_ref)` — 进入应用主循环
- `create_window(...)` — 创建新窗口（tvOS 通常为单窗口）
- `get_window_count()` — 获取窗口数量
- `get_all_monitors()` — 获取所有显示器信息
- `get_drag_and_drop_state()` / `set_drag_and_drop_state()` — 拖放状态管理

**实现细节:**
- 使用 `CADisplayLink` 进行帧同步（与 macOs 的 CVDisplayLink 对应）
- 窗口为 `UIWindow`，通过 `UIViewController` 和 `GCEventViewController` 实现
- 事件处理将 UIKit 触摸事件和 GCEventViewController 的遥控器事件转换为统一事件
- `CVPixelBuffer` 和 Metal 纹理用于渲染
- 使用 `CFRunLoopRunInMode` 驱动主运行循环
- 休眠和空闲管理通过 `UIApplication` 生命周期控制

**平台集成:** 专用 tvOS，使用 UIKit、GameController、CoreGraphics 和 Metal 框架
