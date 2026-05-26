# xlib_window.rs — Xlib Window Implementation

**文件路径**: platform/src/os/linux/x11/xlib_window.rs (1123行)
**核心用途**: 实现 X11 窗口的创建、初始化、几何管理、DPI 获取、IME 输入、最大/最小化、拖拽支持（XDnd）。

## Types/Structs

### `XlibWindow`
X11 窗口结构体。

| 字段 | 类型 | 描述 |
|------|------|------|
| `window` | `Option<c_ulong>` | X11 Window XID |
| `xic` | `Option<XIC>` | X 输入法上下文 |
| `attributes` | `Option<XSetWindowAttributes>` | 窗口属性 |
| `visual_info` | `Option<XVisualInfo>` | 视觉信息 |
| `last_nc_mode` | `Option<c_long>` | 上次的非客户区模式（边缘调整/move） |
| `window_id` | `WindowId` | Makepad 窗口 ID |
| `last_window_geom` | `WindowGeom` | 缓存的上次窗口几何 |
| `ime_spot` | `Vec2d` | IME 光标位置 |
| `current_cursor` | `MouseCursor` | 当前鼠标光标 |
| `last_mouse_pos` | `Vec2d` | 上次鼠标位置 |
| `ime_active` | `bool` | IME 是否激活 |
| `is_popup` | `bool` | 是否为弹出窗口 |
| `popup_parent` | `Option<WindowId>` | 弹出窗口的父窗口 ID |

### `MwmHints`
MOTIF WM Hints 结构（控制窗口装饰）。

| 字段 | 类型 |
|------|------|
| `flags` | `c_ulong` |
| `functions` | `c_ulong` |
| `decorations` | `c_ulong` |
| `input_mode` | `c_long` |
| `status` | `c_ulong` |

### `Dnd`
XDnd 拖拽状态。

| 字段 | 类型 | 描述 |
|------|------|------|
| `atoms` | `DndAtoms` | DnD 相关 Atom |
| `display` | `*mut Display` | X11 显示连接 |
| `type_list` | `Option<Vec<Atom>>` | 源窗口支持的数据类型列表 |
| `selection` | `Option<CString>` | 当前选择数据 |

### `DndAtoms`
XDnd 协议相关 Atom。

| 字段 | Atom 名称 |
|------|-----------|
| `action_private` | `XdndActionPrivate` |
| `aware` | `XdndAware` |
| `drop` | `XdndDrop` |
| `enter` | `XdndEnter` |
| `leave` | `XdndLeave` |
| `none` | `None` |
| `position` | `XdndPosition` |
| `selection` | `XdndSelection` |
| `status` | `XdndStatus` |
| `type_list` | `XdndTypeList` |
| `uri_list` | `text/uri-list` |

## 常量

| 常量 | 值 | 用途 |
|------|-----|------|
| `MWM_HINTS_FUNCTIONS` | `1<<0` | MwmHints 功能标志 |
| `MWM_HINTS_DECORATIONS` | `1<<1` | MwmHints 装饰标志 |
| `_NET_WM_MOVERESIZE_SIZE_TOPLEFT` ~ `_MOVE_KEYBOARD` | 0-10 | 边缘调整/移动方向常量 |
| `_NET_WM_STATE_REMOVE/ADD/TOGGLE` | 0/1/2 | 窗口状态变更操作 |

## Key Methods

### `XlibWindow::new(window_id) -> XlibWindow`
创建空的 XlibWindow 实例。

### `XlibWindow::init(title, size, position, is_fullscreen, visual_info, custom_window_chrome)`
初始化主窗口：
1. 获取 X11 display、root window
2. 设置窗口属性（colormap、event_mask）
3. 通过 `XCreateWindow` 创建 X11 窗口
4. 设置 `WM_PROTOCOLS`（WM_DELETE_WINDOW）
5. 若 `custom_window_chrome` 设置 `_MOTIF_WM_HINTS` 去除装饰
6. 启用 DnD、设置标题、WM_CLASS、窗口图标（`_NET_WM_ICON`）
7. 设置 XSizeHints（含 `USPosition` 确保 WM 遵守位置请求）
8. 创建 XIC（输入法上下文）
9. 发送初始 `WindowGeomChange` 事件

### `XlibWindow::init_popup(parent_window_id, size, position, visual_info)`
初始化弹出窗口：
1. 设置 `override_redirect = 1`（窗口管理器不管理）
2. 设置 `_NET_WM_WINDOW_TYPE_POPUP_MENU`
3. `XMapRaised` 立即显示
4. 创建 XIC

### `XlibWindow::set_x11_icon(display, window)`
通过 `_NET_WM_ICON` 属性设置窗口图标（RGBA8 → ARGB u32 数组）。

### `restore_or_maximize(add_remove)` / `restore()` / `maximize()`
通过 `_NET_WM_STATE` 客户端消息最大化/还原窗口。

### `close_window()` - `XDestroyWindow`
### `minimize()` - `XIconifyWindow`
### `get_window_geom() -> WindowGeom` - 获取完整窗口几何
### `get_is_maximized() -> bool` - 通过 `_NET_WM_STATE` 属性检查
### `get_position() -> Vec2d` - 使用 `XTranslateCoordinates` 获取根窗口坐标
### `get_inner_size() / get_outer_size()` - 通过 `XGetWindowAttributes`
### `set_position(pos)` - `XMoveWindow`
### `get_dpi_factor() -> f64` - 从 X resources 读取 `Xft.dpi`（dpi/96）
### `set_ime_spot(spot)` / `set_ime_active(active)` - IME 位置和激活控制
### `send_change_event()` / `send_focus_event()` / `send_focus_lost_event()` - 事件通知
### `contains_root_pos(root_x, root_y) -> bool` - 根坐标命中检测
### `send_mouse_down/up/move` / `send_text_input` - 鼠标/文本事件

## Dnd (XDnd 协议) 方法
- `enable_for_window(window)` — 设置 `XdndAware` 属性
- `handle_enter_event` — 处理 XDndEnter，获取类型列表
- `handle_drop_event` — 处理 XDndDrop，请求 URI 列表转换
- `handle_leave_event` — 清理类型列表
- `handle_position_event` — 处理 XDndPosition，发送 Status
- `send_status_event` — 发送 XDndStatus 消息
- `convert_selection` — 请求 XdndSelection 转换
- `get_selection_property` / `get_type_list_property` — 读取窗口属性

## Implementation Details
- `USPosition` 标志强制 WM 遵守程序指定的窗口位置
- 双击 caption 触发最大化/还原切换（350ms/5px 阈值）
- 窗口关闭按顺序：EGL 表面 → Wayland/X11 对象
- DnD 支持 XDnd 协议版本 5
