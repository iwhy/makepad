# wayland_state.rs — Wayland Protocol State Management

**文件路径**: platform/src/os/linux/wayland/wayland_state.rs (1675行)
**核心用途**: 管理 Wayland 协议的全部状态，实现所有 Wayland 对象的 Dispatch trait 处理事件分发，包括输入、剪贴板、弹出窗口、键盘等。

## Types/Structs

### `WaylandState`
Wayland 协议的全部运行时状态。

| 字段 | 类型 | 描述 |
|------|------|------|
| `compositor` | `Option<WlCompositor>` | Wayland 合成器 |
| `wm_base` | `Option<XdgWmBase>` | XDG Shell 基协议 |
| `seat` | `Option<WlSeat>` | 输入 Seat |
| `shm` | `Option<WlShm>` | 共享内存 |
| `data_device_manager` | `Option<WlDataDeviceManager>` | 数据设备管理器（剪贴板） |
| `data_device` | `Option<WlDataDevice>` | 数据设备 |
| `clipboard_source` | `Option<WlDataSource>` | 剪贴板数据源 |
| `clipboard_offer` | `Option<ClipboardOffer>` | 当前剪贴板 offer |
| `data_offers` | `Vec<ClipboardOffer>` | 待处理的剪贴板 offers |
| `pending_clipboard_read` | `Option<PendingClipboardRead>` | 待读取的剪贴板数据 |
| `pending_paste_text_input` | `Option<String>` | 待分发的粘贴文本 |
| `pending_clipboard_copy` | `Option<String>` | 等待 serial 的复制内容 |
| `internal_drag_items` | `Option<Arc<Vec<DragItem>>>` | 内部拖拽项 |
| `clipboard_text` | `String` | 剪贴板文本内容 |
| `cursor_manager` | `Option<WpCursorShapeManagerV1>` | 光标形状管理器 |
| `cursor_shape` | `Option<WpCursorShapeDeviceV1>` | 光标形状设备 |
| `pointer` | `Option<WlPointer>` | 指针设备 |
| `last_mouse_pos` | `Vec2d` | 上次鼠标位置 |
| `pointer_serial` | `Option<u32>` | 指针事件序列号 |
| `keyboard_serial` | `Option<u32>` | 键盘事件序列号 |
| `decoration_manager` | `Option<ZxdgDecorationManagerV1>` | 窗口装饰管理器 |
| `icon_manager` | `Option<XdgToplevelIconManagerV1>` | 窗口图标管理器 |
| `windows` | `Vec<WaylandWindow>` | 已创建的窗口 |
| `popups` | `Vec<WaylandPopupWindow>` | 已创建的弹出窗口 |
| `pointer_window` | `Option<WindowId>` | 当前指针所在窗口 |
| `keyboard_window` | `Option<WindowId>` | 当前键盘焦点窗口 |
| `modifiers` | `KeyModifiers` | 当前修饰键状态 |
| `timers` | `SelectTimers` | 定时器管理 |
| `scale_manager` | `Option<WpFractionalScaleManagerV1>` | 分數縮放管理器 |
| `viewporter` | `Option<WpViewporter>` | Viewport 管理器 |
| `xkb_state` | `Option<XkbState>` | XKB 键盘状态 |
| `xkb_cx` | `XkbContext` | XKB 上下文 |
| `text_input` | `Option<ZwpTextInputV3>` | 文本输入协议 |
| `text_input_manager` | `Option<ZwpTextInputManagerV3>` | 文本输入管理器 |
| `primary_selection_manager` | `Option<ZwpPrimarySelectionDeviceManagerV1>` | 主选择管理器 |
| `primary_selection_device` | `Option<ZwpPrimarySelectionDeviceV1>` | 主选择设备 |
| `primary_selection_source` | `Option<ZwpPrimarySelectionSourceV1>` | 主选择数据源 |
| `primary_selection_text` | `String` | 主选择文本 |
| `last_resize_edge` | `Option<ResizeEdge>` | 上次边缘调整方向 |
| `scroll_accumulator` | `Vec2d` | 滚动累积值 |
| `scroll_is_wheel` | `bool` | 是否鼠标滚轮滚动 |
| `last_scroll_time` | `f64` | 上次滚动时间 |
| `event_flow` | `EventFlow` | 事件流程状态 |
| `event_loop_running` | `bool` | 事件循环是否运行 |
| `key_repeat_rate` | `i32` | 键盘重复速率（键/秒） |
| `key_repeat_delay` | `i32` | 键盘重复初始延迟（毫秒） |
| `key_repeat` | `Option<KeyRepeatState>` | 当前重复按键状态 |

### `KeyRepeatState`
按键重复跟踪状态。

| 字段 | 类型 | 描述 |
|------|------|------|
| `key_code` | `KeyCode` | 重复的按键码 |
| `text` | `String` | 重复产生的文本 |
| `in_initial_delay` | `bool` | 是否在初始延迟阶段 |

### `ClipboardOffer`
剪贴板 Offer。

| 字段 | 类型 | 描述 |
|------|------|------|
| `offer` | `WlDataOffer` | 数据 offer |
| `mime_types` | `Vec<String>` | 支持的 MIME 类型列表 |

### `PendingClipboardRead`
待完成的剪贴板读取操作。

| 字段 | 类型 | 描述 |
|------|------|------|
| `fd` | `OwnedFd` | 读取管道的文件描述符 |
| `bytes` | `Vec<u8>` | 已读取的字节数据 |

## Key Dispatch Trait Implementations

| 协议对象 | 主要处理逻辑 |
|---------|-------------|
| `wl_registry::WlRegistry` | 全局对象绑定：compositor, wm_base, seat, data_device_manager, decoration_manager, cursor_shape_manager, scale_manager, viewporter, shm, icon_manager, text_input_manager, primary_selection |
| `xdg_wm_base::XdgWmBase` | 处理 Ping 事件（回应 Pong） |
| `wp_fractional_scale_v1` | 更新窗口 DPI 因子（scale/120） |
| `xdg_toplevel::XdgToplevel` | 处理 Configure（尺寸/最大化/全屏状态）、Close 事件 |
| `xdg_surface::XdgSurface` | 处理 Configure serial，发送首次几何变更事件 |
| `xdg_popup::XdgPopup` | 处理 Configure（位置/尺寸变更）、PopupDone（关闭+关闭原因） |
| `wl_seat::WlSeat` | 处理 Capabilities，创建键盘/指针设备，光标形状设备 |
| `wl_keyboard::WlKeyboard` | 处理 Enter/Leave/Key/Modifiers/RepeatInfo/Keymap 事件 |
| `wl_pointer::WlPointer` | 处理 Enter/Leave/Motion（含边缘调整检测）、Button、Axis/Frame（滚动） |
| `wl_data_device::WlDataDevice` | 处理 DataOffer/Selection 事件 |
| `wl_data_source::WlDataSource` | 处理 Send（通过管道写入剪贴板文本）、Cancelled 事件 |
| `zwp_text_input_v3` | 处理 CommitString 事件（输入法文本提交） |
| `zwp_primary_selection_device_v1` | 创建主选择 offer |
| `zwp_primary_selection_source_v1` | 处理 Send（写入主选择文本）、Cancelled |

## Key Methods

### `WaylandState::new(event_callback)`
构造初始 WaylandState，所有字段设为默认值。

### `window_id_for_surface(surface) -> Option<WindowId>`
通过 wl_surface 查找对应的窗口 ID。

### `xdg_surface_for_window(window_id) -> Option<XdgSurface>`
通过窗口 ID 查找对应的 xdg_surface。

### `set_clipboard_text(qhandle, serial, text)`
设置剪贴板：创建数据源，提供多种 MIME 类型，设置选择。

### `set_primary_selection_text(qhandle, serial, text)`
设置主选择：创建主选择数据源，提供多种文本 MIME 类型。

### `flush_pending_clipboard_copy(qhandle, serial)`
在有可用 serial 时执行延迟的剪贴板复制操作。

### `request_clipboard_paste(conn)`
请求剪贴板粘贴：通过管道接收数据，存入 `pending_paste_text_input`。

### `pump_pending_clipboard_read()`
轮询待处理的剪贴板管道读取，收集完整数据后分发粘贴文本。

### `handle_key_repeat_timer(timer_id) -> bool`
处理键盘重复定时器：
- 初始延迟后切换为稳态重复间隔
- 生成 `KeyDown`（is_repeat=true）和 `TextInput` 事件

### `available() -> bool`
检查 compositor 和 wm_base 是否已绑定就绪。

### `xdg_toplevel_has_state(states, needle) -> bool`
检查 xdg_toplevel 的 states 数组中是否包含指定状态。

### `ensure_data_device(qhandle)` / `ensure_primary_selection_device(qhandle)`
延迟创建 data_device / primary_selection_device。

### `start_internal_drag(items)`
启动内部拖拽。

### `start_timer / stop_timer / time_now`
定时器方法委托给 `SelectTimers`。

## Edge-Resize Detection
在指针 Motion 事件中，检测鼠标是否靠近窗口边缘（5-10px阈值），设置对应的 `ResizeEdge` 和光标形状：

| 区域 | ResizeEdge | 光标形状 |
|------|-----------|---------|
| 左上角 | TopLeft | NwResize |
| 左下角 | BottomLeft | SwResize |
| 左边缘 | Left | WResize |
| 右上角 | TopRight | NeResize |
| 右下角 | BottomRight | SeResize |
| 右边缘 | Right | EResize |
| 上边缘 | Top | NResize |
| 下边缘 | Bottom | SResize |

## Scroll Handling
- `Axis` 事件累积滚动值（取反以统一 Makepad 滚动约定）
- `AxisSource` 区分鼠标滚轮 vs 触摸板
- `Frame` 事件触发实际的 `ScrollEvent`，鼠标滚轮应用加速度曲线

## Popup Dismissal
- 点击主窗口时关闭所有弹出窗口（OutsideClick）
- 键盘焦点离开时关闭弹出窗口（FocusLost）
- 弹出窗口通过 `PopupDone` 关闭（Compositor）
