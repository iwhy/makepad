# xlib_app.rs — Xlib Application Layer

**文件路径**: platform/src/os/linux/x11/xlib_app.rs (1174行)
**核心用途**: 实现 X11 应用层：事件循环、X11 事件分发、窗口管理、剪贴板、选择、鼠标光标设置和拖拽支持。

## Types/Structs

### `XlibApp`
Xlib 应用主结构体。

| 字段 | 类型 | 描述 |
|------|------|------|
| `display` | `*mut Display` | X11 显示连接 |
| `event_loop_running` | `bool` | 事件循环是否运行 |
| `xim` | `XIM` | 输入法管理器 |
| `clipboard` | `String` | 剪贴板文本（CLIPBOARD 选择） |
| `primary_selection` | `String` | 主选择文本（PRIMARY 选择） |
| `display_fd` | `c_int` | 显示连接的文件描述符 |
| `window_map` | `HashMap<c_ulong, *mut XlibWindow>` | XID 到 XlibWindow 的映射 |
| `timers` | `SelectTimers` | 定时器管理 |
| `last_scroll_time` | `f64` | 上次滚动时间 |
| `last_click_time` | `f64` | 上次点击时间 |
| `last_click_pos` | `(i32, i32)` | 上次点击位置 |
| `event_callback` | `Option<Box<...>>` | 事件回调 |
| `event_flow` | `EventFlow` | 事件流状态 |
| `current_cursor` | `MouseCursor` | 当前光标 |
| `internal_cursor` | `MouseCursor` | 内部光标（边缘调整时） |
| `atoms` | `XlibAtoms` | 预获取的 X11 Atom |
| `dnd` | `Dnd` | 拖拽状态 |
| `next_keypress_is_repeat` | `bool` | 下一按键是否为重复 |
| `active_popup` | `Option<c_ulong>` | 活动弹出窗口 XID |
| `active_popup_grabbed_keyboard` | `bool` | 弹出窗口是否抓取了键盘 |

### `XlibAtoms`
预获取的 X11 Atom 集合。

| 字段 | Atom 名称 |
|------|-----------|
| `clipboard` | `CLIPBOARD` |
| `primary` | `PRIMARY` |
| `net_wm_moveresize` | `_NET_WM_MOVERESIZE` |
| `net_wm_icon` | `_NET_WM_ICON` |
| `net_wm_window_type` | `_NET_WM_WINDOW_TYPE` |
| `net_wm_window_type_popup_menu` | `_NET_WM_WINDOW_TYPE_POPUP_MENU` |
| `cardinal` | `CARDINAL` |
| `wm_delete_window` | `WM_DELETE_WINDOW` |
| `wm_protocols` | `WM_PROTOCOLS` |
| `wm_class` | `WM_CLASS` |
| `motif_wm_hints` | `_MOTIF_WM_HINTS` |
| `net_wm_state` | `_NET_WM_STATE` |
| `new_wm_state_maximized_horz` | `_NET_WM_STATE_MAXIMIZED_HORZ` |
| `new_wm_state_maximized_vert` | `_NET_WM_STATE_MAXIMIZED_VERT` |
| `targets` | `TARGETS` |
| `string` | `STRING` |
| `utf8_string` | `UTF8_STRING` |
| `text` | `TEXT` |
| `text_plain` | `text/plain` |
| `multiple` | `MULTIPLE` |
| `atom` | `ATOM` |

### 全局函数
- `get_xlib_app_global()` — 获取 XlibApp 全局单例
- `init_xlib_app_global(event_callback)` — 初始化 XlibApp 全局单例

## Key Methods

### `XlibApp::new(event_callback)`
构造 XlibApp：打开 X11 显示、获取文件描述符、设置 locale、打开 XIM、初始化 Xrm、创建 Atoms。

### `event_loop_poll()`
单次事件轮询：
1. 按优先级枚举 `XPending` 事件，通过 `XNextEvent` 获取
2. 对每个事件类型进行分发

#### 事件处理表

| X11 事件 | 处理方式 |
|---------|---------|
| `SelectionNotify` | 读取选择的属性数据 → `TextInput(was_paste=true)` |
| `SelectionRequest` | 向请求者发送剪贴板/主选择文本 |
| `DestroyNotify` | 清理 window_map，发送 `WindowClosed` |
| `ConfigureNotify` | 检查窗口变更 → `send_change_event` |
| `MotionNotify` | 鼠标移动 → 边缘调整检测(8方向)、caption 检测、转发 `MouseMove` |
| `ButtonPress` | 按钮 4-7 为滚轮(加速度曲线)，活动弹窗外点击 → `PopupDismissed`，正常点击 → `MouseDown`，双击 caption→ 切换最大化 |
| `ButtonRelease` | → `MouseUp` |
| `KeyPress` | Ctrl+V/C/X 处理剪贴板，→ `KeyDown`，XIM `Xutf8LookupString` → `TextInput` |
| `KeyRelease` | 检测键盘重复，→ `KeyUp` |
| `ClientMessage` | WM_PROTOCOLS → 关闭窗口，XDnd enter/drop/leave/position |
| `Expose` | 忽略（EGL 处理重绘） |
| `VisibilityNotify` | 非 obscured → `send_focus_event` |
| `FocusIn` | 设置 XIC focus → `send_focus_event` |
| `FocusOut` | 弹窗 FocusLost → `PopupDismissed`，取消 XIC focus |

### `event_loop()`
主事件循环：Wait/Poll/Exit 状态机，与 `WaylandApp::event_loop` 相同模式。

### `do_callback(event)`
取出回调函数，调用后检查 Exit 状态。

### `terminate_event_loop()`
关闭 XIM 和 X11 显示连接。

### `activate_popup_grab(popup_window, grab_keyboard)` / `release_popup_grab(popup_window)`
激活/释放弹出窗口的指针（和可选的键盘）抓取。

### `set_mouse_cursor / restore_mouse_cursor / set_internal_mouse_cursor`
鼠标光标设置：通过 `XcursorLibraryLoadCursor` 加载光标主题，通过 `XDefineCursor` 应用到所有窗口。

### `xbutton_to_mouse_button(button) -> MouseButton`
X11 按钮编号到 Makepad MouseButton 的转换。

### `xkeystate_to_modifiers(state) -> KeyModifiers`
X11 键盘状态掩码到 Makepad KeyModifiers 的转换。

### `xkeyevent_to_keycode(key_event) -> KeyCode`
通过 `XLookupString` 获取 keysym，映射到 Makepad KeyCode。

### `copy_to_clipboard(text, window_id, time)` / `set_primary_selection(text, window_id, time)`
设置 `XSetSelectionOwner` 拥有 CLIPBOARD/PRIMARY 选择。

## Edge-Resize Detection (MotionNotify)
与 Wayland 后端相同的边缘检测算法：
- 8 个方向 + Move（caption）模式
- 通过 `_NET_WM_MOVERESIZE` 客户端消息触发窗口管理器调整
