# win32_event.rs — Win32 平台事件枚举

**文件路径**: platform/src/os/windows/win32_event.rs (40行)
**核心用途**: 定义 Win32 平台特定的事件枚举 `Win32Event`，作为 Win32 消息到 Makepad 事件系统的桥梁。

## 枚举

### `Win32Event`
```rust
pub enum Win32Event {
    WindowGotFocus(WindowId),           // 窗口获得焦点
    WindowLostFocus(WindowId),          // 窗口失去焦点
    WindowResizeLoopStart(WindowId),    // 窗口大小调整开始（拖动边框）
    WindowResizeLoopStop(WindowId),     // 窗口大小调整结束
    WindowGeomChange(WindowGeomChangeEvent),  // 窗口几何变更
    WindowClosed(WindowClosedEvent),    // 窗口关闭
    PopupDismissed(PopupDismissedEvent),// 弹出窗口关闭
    Paint,                              // 绘制请求

    // 鼠标事件
    MouseDown(MouseDownEvent),
    MouseUp(MouseUpEvent),
    MouseMove(MouseMoveEvent),
    MouseLeave(MouseLeaveEvent),
    Scroll(ScrollEvent),

    // 窗口操作
    WindowDragQuery(WindowDragQueryEvent),  // 拖放查询
    WindowCloseRequested(WindowCloseRequestedEvent), // 关闭请求（Alt+F4）
    TextInput(TextInputEvent),          // 文本输入
    
    // 拖放
    Drag(DragEvent),                    // 拖入/悬停
    Drop(DropEvent),                    // 放下
    DragEnd,                            // 拖放结束

    // 键盘
    KeyDown(KeyEvent),
    KeyUp(KeyEvent),
    TextCopy(TextClipboardEvent),       // 复制（Ctrl+C）
    TextCut(TextClipboardEvent),        // 剪切（Ctrl+X）

    // 计时器
    Timer(TimerEvent),

    // 信号（跨线程通知）
    Signal,
}
```

## 平台集成

- 由 `win32_app.rs` 中的窗口消息处理（`WndProc`）生成
- 在 `windows.rs` 的 `win32_event_callback` 中处理，转换为 Makepad `Event`
- `Signal` 用于跨线程通信（Media Foundation、WASAPI、MIDI 等设备变更通知）
- `DragEnd` 触发模拟的 `MouseUp` 事件以清除拖放状态
