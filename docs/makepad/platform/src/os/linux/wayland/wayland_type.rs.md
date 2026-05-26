# wayland_type.rs — Cursor and Mouse Button Type Conversions

**文件路径**: platform/src/os/linux/wayland/wayland_type.rs (48行)
**核心用途**: 定义 Makepad 鼠标光标和按钮类型与 Wayland 协议类型之间的转换函数。

## 类型转换

### `MouseCursor -> wp_cursor_shape_device_v1::Shape`
将 Makepad 内部 `MouseCursor` 枚举映射到 Wayland cursor-shape 协议的 `Shape` 枚举：

| MouseCursor | Shape |
|-------------|-------|
| `Hidden` | `Default` |
| `Default` | `Default` |
| `Crosshair` | `Crosshair` |
| `Hand` | `Pointer` |
| `Arrow` | `ContextMenu` |
| `Move` | `Move` |
| `Text` | `Text` |
| `Wait` | `Wait` |
| `Help` | `Help` |
| `NotAllowed` | `NotAllowed` |
| `Grab` | `Grab` |
| `Grabbing` | `Grabbing` |
| 8个方向调整 | 对应 N/Ne/E/Se/S/Sw/W/Nw Resize |
| `NsResize` | `NsResize` |
| `NeswResize` | `NeswResize` |
| `EwResize` | `EwResize` |
| `NwseResize` | `NwseResize` |
| `ColResize` | `ColResize` |
| `RowResize` | `RowResize` |

## Key Functions

### `from_mouse(button: u32) -> Option<MouseButton>`
将 Linux 输入事件码（`linux/input-event-codes.h`）转换为 Makepad 鼠标按钮：

| 值 | 按钮 |
|----|------|
| `0x110` | PRIMARY（左键） |
| `0x111` | SECONDARY（右键） |
| `0x112` | MIDDLE（中键） |
| `0x116` | BACK（后退） |
| `0x117` | FORWARD（前进） |
