# ios_event.rs — iOS 平台事件枚举

**文件路径**: `platform/src/os/apple/ios/ios_event.rs` (41 行)
**核心作用**: 定义 `IosEvent` 枚举，封装 iOS 原生事件（生命周期、触摸、键盘、文本、定时器等），作为 UIKit 回调到 Makepad 事件系统的桥梁。

## IosEvent 枚举

```rust
pub enum IosEvent {
    Init, Foreground, Background, Pause, Resume, Shutdown,    // 应用生命周期
    WindowGotFocus(WindowId), WindowLostFocus(WindowId),       // 窗口焦点
    WindowGeomChange(WindowGeomChangeEvent),                   // 窗口几何变化
    Paint,                                                     // 绘制请求
    VirtualKeyboard(VirtualKeyboardEvent),                     // 虚拟键盘事件
    MouseDown(MouseDownEvent), MouseUp(MouseUpEvent),          // 鼠标（触控笔）事件
    MouseMove(MouseMoveEvent),
    TouchUpdate(TouchUpdateEvent),                             // 多点触摸更新
    LongPress(LongPressEvent),                                 // 长按手势
    Scroll(ScrollEvent),                                       // 滚动事件
    TextInput(TextInputEvent),                                 // 文本输入
    TextRangeReplace(TextRangeReplaceEvent),                   // 文本范围替换
    SelectionHandleDrag(SelectionHandleDragEvent),             // 选择手柄拖拽
    KeyDown(KeyEvent), KeyUp(KeyEvent),                        // 键盘事件
    TextCopy(TextClipboardEvent), TextCut(TextClipboardEvent), // 剪贴板
    Timer(TimerEvent),                                         // 定时器
    PermissionResult(PermissionResult),                        // 权限请求结果
}
```

## 事件分类

1. **应用生命周期**: `Init` → `Foreground/Background` → `Pause/Resume` → `Shutdown`
2. **窗口**: 焦点变化、几何变化（旋转/分屏）
3. **交互**: 触摸（多点）、鼠标（触控笔）、长按、滚动
4. **文本输入**: 普通文本、IME 范围替换、选择手柄
5. **键盘**: 物理键盘按键（硬件键盘）
6. **系统**: 定时器、绘制、权限结果

## 与 iOS 系统集成

- `IosEvent` 是 UIKit 回调（`ios_delegates.rs`）或应用层（`ios_app.rs`）构造的原始事件格式
- 通过 `ios.rs` 中的 `ios_event_callback` 方法分派到 Makepad 标准事件系统
- 所有坐标在分派前由 `native_*_to_layout` 转换从 UIKit 点坐标到 Makepad 布局坐标
