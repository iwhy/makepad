# ios_delegates.rs — UIKit 委托与自定义 Objective-C 类定义

**文件路径**: `platform/src/os/apple/ios/ios_delegates.rs` (790 行)
**核心作用**: 使用 Rust 的 Objective-C 运行时 API（`objc` 库）动态注册 UIKit 所需的 delegate 类和子类，处理触摸事件、绘制、键盘通知、手势识别、剪贴板菜单和编辑菜单定位。

## 重入安全辅助

### `try_with_ios_app(f)`

避免重入 borrow panic 的包装器。在 UIKit 回调中，如果当前已处于 `with_ios_app` 调用中，`try_borrow_mut()` 会返回 `Err`，返回 `None` 而非 panic。

## 自定义 Objective-C 类

### MakepadViewController (`define_makepad_view_controller`)

继承 `UIViewController`。

| ivar | 类型 | 说明 |
|------|------|------|
| `_prefersStatusBarHidden` | `BOOL` | 状态栏隐藏 |
| `_prefersHomeIndicatorAutoHidden` | `BOOL` | Home Indicator 隐藏 |

方法：
- `prefersStatusBarHidden` / `prefersHomeIndicatorAutoHidden` — 返回 ivar 值
- `viewSafeAreaInsetsDidChange` — 安全区域变化时调用 `IosApp::check_window_geom()`，确保设备旋转后几何信息正确

### NSAppDelegate (`define_ios_app_delegate`)

继承 `NSObject`，作为 `UIApplicationDelegate`。

| 回调 | 触发事件 |
|------|----------|
| `application:didFinishLaunchingWithOptions:` | 调用 `app.did_finish_launching_with_options()`，返回 `YES` |
| `applicationWillEnterForeground:` | `IosEvent::Foreground` |
| `applicationDidEnterBackground:` | `IosEvent::Background` |
| `applicationWillResignActive:` | `IosEvent::Pause` |
| `applicationDidBecomeActive:` | `IosEvent::Resume` |
| `applicationWillTerminate:` | `IosEvent::Shutdown` |

### MakepadView (`define_mtk_view`)

继承 `MTKView`，自定义触摸和剪贴板处理。

| ivar | 类型 | 说明 |
|------|------|------|
| `has_selection` | `BOOL` | 是否有文本选择 |
| `menu_rect_x/y/width/height` | `f64` | 编辑菜单位置矩形 |
| `edit_menu_interaction` | `*mut c_void` | 编辑菜单交互引用 |

关键方法：
- `isOpaque` → `YES`（不透明优化）
- `touchesBegan/Moved/Ended/Canceled:withEvent:` — 遍历所有 UITouch，读取 `estimationUpdateIndex` 作为 UID、`locationInView` 作为位置、`majorRadius` 作为半径、`force` 作为力度，调用 `IosApp::send_touch_update()`
- `canBecomeFirstResponder` → `YES`（支持剪贴板菜单）
- `inputView` → `nil`（阻止键盘弹出，由隐藏的 UITextInput 视图处理）
- `canPerformAction:withSender:` — 根据 `has_selection` 和剪贴板状态过滤 `copy:/cut:/paste:/selectAll:` 操作
- `copy:/cut:/paste:/selectAll:` — 调用 `IosApp::send_clipboard_action`/`send_clipboard_paste`

### MakepadViewDlg (`define_mtk_view_delegate`)

继承 `NSObject`，作为 `MTKViewDelegate`。

| ivar | 类型 | 说明 |
|------|------|------|
| `display_ptr` | `*mut c_void` | 显示指针 |

方法：
- `drawInMTKView:` — 调用 `IosApp::draw_in_rect()`
- `mtkView:drawableSizeWillChange:` — 调用 `IosApp::draw_size_will_change(view, size)`

### LongPressGestureRecognizerHandler (`define_gesture_recognizer_handler`)

继承 `NSObject`，作为长按手势的目标。

- `handleLongPressGesture:gestureRecognizer:` — 当手势状态变为 `Began`（state == 1）时，获取视图中的位置并调用 `IosApp::send_long_press`

### SelectionHandlePanRecognizerHandler (`define_selection_handle_gesture_handler`)

继承 `NSObject`，作为选择手柄拖拽手势的目标。

| ivar | 类型 | 说明 |
|------|------|------|
| `handle_kind` | `i64` | 0 = Start, 1 = End |

- `handleSelectionHandlePan:` — 根据手势状态（Began/Changed/Ended/Cancelled/Failed）转换为 `SelectionHandlePhase`，根据 ivar `handle_kind` 确定拖拽的是哪个手柄，获取在 host view 中的位置，保持手柄在手指下方并调用 `IosApp::send_selection_handle_drag`

### TimerDelegate (`define_ios_timer_delegate`)

继承 `NSObject`。

- `receivedTimer:` — 调用 `IosApp::send_timer_received(nstimer)`
- `receivedLiveResize:` — 调用 `IosApp::send_paint_event()`

### NSTextFieldDlg (`define_textfield_delegate`)

继承 `NSObject`，作为键盘通知观察者。

6 个键盘通知处理方法，使用 `keyboardWillChangeFrame` 作为**单一事实源**：
- **`keyboardWillChangeFrame:`** — 计算键盘几何（可见性、底部重叠高度），提取动画曲线和持续时间，排队 `WillShow`/`WillHide` 事件
- **`keyboardDidChangeFrame:`** — 排队 `DidShow`/`DidHide` 事件
- `keyboardWillShow/DidShow/WillHide/DidHide:` — 空操作，不做重复处理
- `inputModeDidChange:` — 键盘语言变化时调用 `reloadInputViews` 以重新查询 `autocorrectionType`

键盘几何计算 `get_keyboard_geometry_in_view(notif, view)`:
1. 从 `UIKeyboardFrameEndUserInfoKey` 获取结束帧（屏幕坐标）
2. 通过屏幕 coordinate space 转换到视图坐标（正确处理 iPad 分屏/拖放）
3. 计算可见区域和键盘底部与视图底部的重叠量
4. 检测是否 docked（底部对齐），浮动键盘返回 0 重叠

动画曲线转换 `get_curve_duration(notif)`:
- UIKit 动画曲线（0-3）映射到 `Ease::Bezier` 三次贝塞尔曲线
- 处理 iOS 7+ 的曲线值编码（`curve > 3` 时右移 16 位）

### MakepadEditMenuDelegate (`define_edit_menu_interaction_delegate`)

继承 `NSObject`，作为 `UIEditMenuInteractionDelegate`。

| ivar | 类型 | 说明 |
|------|------|------|
| `mtk_view` | `*mut c_void` | MTKView 引用 |

方法：
- `editMenuInteraction:targetRectForConfiguration:` — 从 MTKView 的 ivar 读取 `menu_rect_x/y/width/height` 返回菜单目标矩形
