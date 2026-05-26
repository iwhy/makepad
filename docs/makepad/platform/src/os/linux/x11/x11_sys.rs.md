# x11_sys.rs — X11 Library FFI Bindings

**文件路径**: platform/src/os/linux/x11/x11_sys.rs (1228行)
**核心用途**: 提供 X11 库(libX11)、Xcursor 和 C 标准库(setlocale)的完整 FFI 绑定，包括类型定义、常量和数据结构。

## 类型别名
| 别名 | 原始类型 | 描述 |
|------|---------|------|
| `Display` | `_XDisplay` | X11 显示连接 |
| `Window / XID / Drawable` | `c_ulong` | X 资源标识符 |
| `Colormap / KeySym / Pixmap / Cursor` | `c_ulong` | 各种 X 资源类型 |
| `Time` | `c_ulong` | 时间戳 |
| `XIM` | `*mut _XIM` | 输入法管理器 |
| `XIC` | `*mut _XIC` | 输入法上下文 |
| `Atom` | `c_ulong` | 属性原子 |
| `XEvent` | `_XEvent` | X 事件联合体 |
| `VisualID` | `c_ulong` | Visual ID |

## 常量 (部分重要)
**事件类型**: `SelectionNotify(31)`, `ButtonPress(4)`, `ButtonRelease(5)`, `KeyPress(2)`, `KeyRelease(3)`, `MotionNotify(6)`, `Expose(12)`, `DestroyNotify(17)`, `ConfigureNotify(22)`, `ClientMessage(33)`, `FocusIn(9)`, `FocusOut(10)`, `VisibilityNotify(15)` 等。

**事件掩码**: `ExposureMask(32768)`, `StructureNotifyMask(131072)`, `PointerMotionMask(64)`, `ButtonPressMask(4)`, `KeyPressMask(1)` 等。

**修饰键掩码**: `ShiftMask(1)`, `ControlMask(4)`, `Mod1Mask(8)`, `Mod4Mask(64)`。

**XLookup 状态**: `XLookupNone(1)`, `XLookupChars(2)`, `XLookupKeySym(3)`, `XLookupBoth(4)`。

**Keysym 常量**: 所有标准 `XK_*` 常量（a-z, A-Z, 0-9, F1-F12, 编辑键, 数字键盘, 修饰键等），完整的 Keysym 值列表在文件中定义。

## FFI 函数 (libX11)

| 函数 | 用途 |
|------|------|
| `XOpenDisplay / XCloseDisplay` | 打开/关闭显示连接 |
| `XConnectionNumber` | 获取显示连接 fd |
| `XOpenIM / XCloseIM` | 打开/关闭输入法 |
| `XInternAtom` | 获取 Atom |
| `XPending / XNextEvent / XPeekEvent / XEventsQueued` | 事件队列管理 |
| `XGetWindowProperty / XChangeProperty / XFree` | 窗口属性读写 |
| `XSendEvent` | 发送客户端消息 |
| `XDefaultScreen / XRootWindow` | 获取默认屏幕/根窗口 |
| `XGetVisualInfo / XCreateWindow / XDestroyWindow` | Visual 和窗口创建/销毁 |
| `XMapWindow / XMapRaised` | 映射窗口到屏幕 |
| `XMoveWindow / XFlush` | 移动窗口 |
| `XSetWMProtocols` | 设置 WM 协议 |
| `Xutf8SetWMProperties` | 设置 UTF-8 WM 属性 |
| `XCreateIC / XSetICFocus / XUnsetICFocus / XSetICValues` | 输入法上下文管理 |
| `XSetLocaleModifiers` | 设置 locale 修饰符 |
| `XFilterEvent` | 输入法事件过滤 |
| `XGetWindowAttributes` | 获取窗口属性 |
| `XTranslateCoordinates` | 坐标转换 |
| `XResourceManagerString / XrmGetStringDatabase / XrmGetResource` | X Resources |
| `XConvertSelection / XSetSelectionOwner` | 选择(剪贴板)管理 |
| `XSetInputFocus` | 设置输入焦点 |
| `XGrabPointer / XUngrabPointer` | 指针抓取 |
| `XGrabKeyboard / XUngrabKeyboard` | 键盘抓取 |
| `XDefineCursor / XFreeCursor` | 光标管理 |
| `XLookupString / Xutf8LookupString` | 按键事件 → keysym |
| `XIconifyWindow` | 最小化窗口 |
| `XVaCreateNestedList` | 变参列表创建 |
| `XSetLocaleModifiers` | locale 设置 |

### FFI 函数 (Xcursor)
| 函数 | 用途 |
|------|------|
| `XcursorLibraryLoadCursor` | 从光标主题加载光标 |

### FFI 函数 (libc)
| 函数 | 用途 |
|------|------|
| `setlocale` | 设置 C locale |

## 数据结构

**核心 X11 结构体**:
- `_XEvent` — 事件联合体（包含所有事件类型成员）
- `XWindowAttributes` — 窗口属性（位置、尺寸、depth、visual 等）
- `Visual` — 视觉信息（visualid、掩码、bits_per_rgb）
- `XSetWindowAttributes` — 窗口创建属性
- `XVisualInfo` — Visual 信息结构
- `XKeyEvent / XButtonEvent / XMotionEvent / XCrossingEvent` — 输入事件
- `XFocusChangeEvent / XExposeEvent / XVisibilityEvent` — 窗口状态事件
- `XConfigureEvent / XDestroyWindowEvent` — 结构事件
- `XSelectionEvent / XSelectionRequestEvent` — 选择事件
- `XClientMessageEvent` — 客户端消息事件（含 data 联合体）
- `XSizeHints / XWMHints / XClassHint` — WM 提示
- `Screen / Depth` — 屏幕/深度信息
- `XrmValue` — X Resources 值
- 另有 ~20 个其他 X11 事件结构体定义
