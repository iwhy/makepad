# xlib_event.rs — XlibEvent Enum Definition

**文件路径**: platform/src/os/linux/x11/xlib_event.rs (35行)
**核心用途**: 定义 XlibEvent 枚举，这是 X11 和 Wayland 后端共享的统一事件抽象层。

## Enum: `XlibEvent`

| 变体 | 携带数据 | 描述 |
|------|---------|------|
| `WindowGotFocus` | `WindowId` | 窗口获得焦点 |
| `WindowLostFocus` | `WindowId` | 窗口失去焦点 |
| `WindowGeomChange` | `WindowGeomChangeEvent` | 窗口几何/尺寸/DPI 变更 |
| `WindowClosed` | `WindowClosedEvent` | 窗口关闭 |
| `PopupDismissed` | `PopupDismissedEvent` | 弹出窗口关闭 |
| `Paint` | — | 请求绘制帧 |
| `MouseDown` | `MouseDownEvent` | 鼠标按下 |
| `MouseUp` | `MouseUpEvent` | 鼠标释放 |
| `MouseMove` | `MouseMoveEvent` | 鼠标移动 |
| `Scroll` | `ScrollEvent` | 滚动事件 |
| `WindowDragQuery` | `WindowDragQueryEvent` | 窗口拖拽查询 |
| `WindowCloseRequested` | `WindowCloseRequestedEvent` | 窗口关闭请求 |
| `TextInput` | `TextInputEvent` | 文本输入 |
| `Drag` | `DragEvent` | 拖拽中 |
| `Drop` | `DropEvent` | 拖拽放下 |
| `DragEnd` | — | 拖拽结束 |
| `KeyDown` | `KeyEvent` | 按键按下 |
| `KeyUp` | `KeyEvent` | 按键释放 |
| `TextCopy` | `TextClipboardEvent` | 剪贴板复制请求 |
| `TextCut` | `TextClipboardEvent` | 剪贴板剪切请求 |
| `Timer` | `TimerEvent` | 定时器事件 |

## Implementation Details
`XlibEvent` 作为跨后端（X11 和 Wayland）的公共事件类型，使两者的事件处理逻辑能够复用。所有变体都派生 `Debug`。
