# ios_app.rs — iOS 应用生命周期与全局状态

**文件路径**: `platform/src/os/apple/ios/ios_app.rs` (1572 行)
**核心作用**: 管理 iOS 应用的全局单例 `IosApp`，包含 UIKit 窗口/视图初始化、IME 文本输入、剪贴板、摄像头预览层、选择手柄、全屏控制等功能。

## 全局状态

### IosApp 结构体

核心字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `time_start` | `Instant` | 应用启动时间戳 |
| `mtk_view` | `Option<ObjcId>` | MTKView（Metal 渲染视图） |
| `text_input_view` | `Option<ObjcId>` | UITextInput 视图（IME） |
| `view_controller` | `Option<ObjcId>` | MakepadViewController |
| `event_callback` | `Box<dyn FnMut(IosEvent) -> EventFlow>` | 事件回调 |
| `timers` | `Vec<IosTimer>` | 活跃 NSTimer 列表 |
| `touches` | `Vec<TouchPoint>` | 当前触摸点 |
| `camera_preview_layers` | `HashMap<u64, ObjcId>` | 原生摄像头预览层 |
| `selection_handle_start_view/end_view` | `Option<ObjcId>` | 自定义选择手柄视图 |
| `native_selection_display_interaction` | `Option<ObjcId>` | iOS 16+ 原生选择交互 |
| `edit_menu_interaction` | `Option<ObjcId>` | iOS 16+ 编辑菜单交互 |
| `keyboard_observer_delegate` | `Option<ObjcId>` | 键盘通知观察者 |
| `queued_text_events` | `Vec<IosTextInputEvent>` | 文本事件队列（避免重入） |
| `ime_position` | `Option<DVec2>` | IME 候选窗口位置 |
| `last_keyboard_config` | `Option<TextInputConfig>` | 缓存键盘配置 |

### IosClasses 结构体

注册所有自定义 Objective-C 类的指针集合：

- `app_delegate`, `view_controller`, `mtk_view`, `mtk_view_delegate`
- `gesture_recognizer_handler`, `selection_handle_gesture_handler`
- `textfield_delegate`, `timer_delegate`, `edit_menu_delegate`
- `text_position`, `text_range`, `text_selection_rect`, `text_input_view`

### 全局变量

```rust
pub static mut IOS_CLASSES: *const IosClasses       // 线程安全，多线程读取
thread_local! { pub static IOS_APP: RefCell<Option<IosApp>> }  // 线程本地应用实例
```

### 访问辅助

- `with_ios_app(f)` — 借用可变引用调用闭包
- `init_ios_app_global(device, callback)` — 初始化全局类和应用
- `get_ios_class_global()` — 获取 IosClasses 引用

## 关键方法

### 应用生命周期

- `new(metal_device, event_callback)` — 创建 IosApp，初始化 pasteboard 和 edit menu delegate
- `did_finish_launching_with_options()` — 完整的 UIKit 视图初始化：
  1. 创建 UIWindow 和 MTKView（120fps 渲染）
  2. 配置长按手势识别器（`cancelsTouchesInView = NO`）
  3. 创建 MakepadViewController 并设置为 root
  4. 创建 UITextInput 视图（1x1 点大小）并添加为 MTKView 子视图
  5. 初始化 markedText、cursorPosition、selectionStart/End 等 ivar
  6. **iOS 16+**: 尝试使用 `UITextSelectionDisplayInteraction`
  7. 创建自定义选择手柄视图（UIView + UIPanGestureRecognizer）
  8. 注册键盘通知观察者（6 个 NSNotification）
  9. 创建 UIEditMenuInteraction（iOS 16+）或 UIMenuController 备用方案
- `event_loop()` — 调用 `UIApplicationMain` 启动 UIKit 事件循环

### 窗口几何管理

- `draw_size_will_change(view, size)` — MTKView drawable 大小即将变化时的回调。使用 `contentScaleFactor` 计算逻辑点大小，从 `safeAreaInsets` 获取安全区域
- `check_window_geom()` — 从 MTKView bounds 读取当前几何确保与触摸坐标一致，回退到 UIScreen
- `apply_new_window_geom(inner_size, dpi, safe_insets)` — 构造 WindowGeom 并触发 Init（首次）或 WindowGeomChange 事件
- `draw_in_rect()` — 检查几何、标记 first_draw=false、发送 Paint 事件

### 触摸处理

- `update_touch(uid, abs, state)` / `update_touch_with_details(...)` — 更新/添加触摸点
- `send_touch_update()` — 克隆当前触摸、发送 TouchUpdate 事件、移除已结束触摸
- `send_long_press(abs, uid)` — 发送长按事件

### 文本输入与 IME

- `configure_keyboard(config)` — 设置键盘类型、自动大写、自动纠正、返回键类型、安全文本，缓存配置避免重复调用 `reloadInputViews`
- `show_keyboard()` / `hide_keyboard()` — 通过 `becomeFirstResponder`/`resignFirstResponder` 控制键盘展示，注意重入问题
- `set_ime_position(pos)` — 设置 IME 候选窗口位置（同时写入 ivar 避免重入）
- `set_ime_text(text, selection_start, selection_end)` — 从 Rust 同步文本/选择到 UITextInput 缓冲区，使用 UTF-16 索引，通过 inputDelegate 发送 `textWillChange`/`textDidChange` 通知
- 事件队列方法：`send_text_input`, `send_text_range_replace`, `send_text_selection_changed`, `send_backspace`, `send_return_key`

### 键盘类型常量

- `UI_KEYBOARD_TYPE_DEFAULT..=UI_KEYBOARD_TYPE_ASCII_CAPABLE_NUMBER_PAD` (12 种)
- `UI_TEXT_AUTOCAPITALIZATION_NONE..=UI_TEXT_AUTOCAPITALIZATION_ALL`
- `UI_TEXT_AUTOCORRECTION_DEFAULT..=UI_TEXT_AUTOCORRECTION_YES`
- `UI_RETURN_KEY_DEFAULT..=UI_RETURN_KEY_CONTINUE`

### 剪贴板

- `show_clipboard_actions(has_selection, rect, keyboard_shift)` — 显示编辑菜单（iOS 16+ UIEditMenuInteraction 或 iOS 15 UIMenuController 回退）
- `hide_clipboard_actions()` — 隐藏编辑菜单
- `send_clipboard_action("copy"/"cut"/"select_all")` — 分发剪贴板操作，发送 TextCopy/TextCut 事件或模拟 Cmd+A 键盘事件
- `send_clipboard_paste()` — 从系统剪贴板读取并发送 TextInput 事件

### 选择手柄

iOS 提供了两层实现：
1. **iOS 16+**: `UITextSelectionDisplayInteraction` 原生选择显示
2. **自定义回退**: 两个 UIView 子视图 + UIPanGestureRecognizer

- `show_selection_handles(start, end)` / `update_selection_handles(start, end)` / `hide_selection_handles()`
- `update_native_selection_display(start, end, visible)` — 设置 ivar 并通过 inputDelegate 通知
- `send_selection_handle_drag(handle, phase, abs)` — 发送选择手柄拖拽事件

### 摄像头预览

- `attach_camera_preview(video_id, session)` — 创建 `AVCaptureVideoPreviewLayer` 添加到 MTKView 的 superview layer
- `update_camera_preview(video_id, rect, visible)` — 更新位置/可见性
- `detach_camera_preview(video_id)` — 从父层移除

### 定时器

- `start_timer(timer_id, interval, repeats)` — 创建 NSTimer 添加到主 run loop
- `stop_timer(timer_id)` — 使 NSTimer 失效
- `send_timer_received(nstimer)` — 非重复定时器自动移除并发送 Timer 事件

### 其他

- `set_fullscreen(fullscreen)` — 设置状态栏和 Home Indicator 隐藏
- `copy_to_clipboard(content)` / `paste_from_clipboard()` — 使用 UIPasteboard
- `get_ios_directory_paths()` — 获取 NSApplicationSupportDirectory 路径
- `create_selection_handle_view()` — 创建 24x24 蓝色圆形 UIView 作为手柄

## 重入保护模式

整个文件遵循一个关键模式：**先提取 UI 对象指针释放 borrow，再进行 UIKit 调用**。因为 UIKit 方法（`becomeFirstResponder`、`reloadInputViews`、`addSubview` 等）可能同步触发回调（键盘通知、布局变化），这些回调会尝试借用 `IOS_APP`，如果仍在 borrow 中会导致 panic。

```rust
// 典型模式
let view = IOS_APP.try_with(|app| {
    app.try_borrow().ok().and_then(|app_ref| app_ref.as_ref()?.text_input_view)
}).ok().flatten();

// 调用 UIKit（borrow 已释放）
if let Some(v) = view {
    let () = unsafe { msg_send![v, becomeFirstResponder] };
}
```

## IosTextInputEvent 枚举

```rust
pub enum IosTextInputEvent {
    TextInput(String, bool),                        // 文本输入（内容, replace_last）
    RangeReplace(usize, usize, String),             // 范围替换（start, end, text）
    SelectionChanged(String, usize, usize),          // 选择变化（text, start, end）
    KeyEvent(KeyCode),                              // 按键事件
}
```

事件通过 `queued_text_events: Vec<IosTextInputEvent>` 队列暂存，在下一个 timer tick 批量处理，避免 UITextInput 回调重入。
