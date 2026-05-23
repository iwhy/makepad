# `window.rs` — 窗口事件类型

## 概述

该文件定义了与窗口状态变更相关的数据结构，包括：**安全区（Safe Area）描述**、**窗口几何信息**、**几何变更/移动/关闭事件**、**弹窗关闭事件**以及**窗口拖拽查询**事件。

---

## `SafeAreaInsets` — 安全区域描述

描述屏幕上不应放置交互内容的区域（如刘海/灵动岛、Home Indicator、圆角）：

```rust
pub struct SafeAreaInsets {
    pub top: f64,
    pub right: f64,
    pub bottom: f64,
    pub left: f64,
}
```

### `SafeAreaInsets::scale(self, scale) -> Self`
将四个方向的安全区域值乘以给定的缩放系数。平台后端收到原始物理像素后需通过 `CxWindow` 转换，最终以 Makepad 布局点为单位发布。

---

## `WindowGeom` — 窗口几何信息

聚合窗口的所有空间属性：

| 字段 | 类型 | 描述 |
|------|------|------|
| `dpi_factor` | `f64` | 有效 DPI 因子（OS DPI 或用户覆盖值） |
| `can_fullscreen` | `bool` | 是否可全屏 |
| `xr_is_presenting` | `bool` | XR 是否正在展示 |
| `is_fullscreen` | `bool` | 是否全屏 |
| `is_topmost` | `bool` | 是否置顶 |
| `position` | `Vec2d` | 窗口位置 |
| `inner_size` | `Vec2d` | 窗口内容区尺寸 |
| `outer_size` | `Vec2d` | 窗口外沿尺寸（含边框） |
| `safe_area_insets` | `SafeAreaInsets` | 安全区域 |
| `window_chrome_buttons` | `Rect` | 窗口装饰按钮占位矩形 |

### `window_chrome_buttons` 跨平台行为

此字段描述平台窗口按钮（关闭/最小化/最大化）在自定义标题栏中的占位区域：

| 平台 | 行为 |
|------|------|
| **macOS** | 从 OS 实时查询三个交通灯按钮的包围盒，位于标题栏左侧 |
| **Windows** | Makepad 自绘的三个按钮，每个 46×29 逻辑像素，右对齐 |
| **Linux/Wayland** | 同 Windows（138×29 逻辑像素），右对齐 |
| **X11/Android/iOS/Web** | `Rect::default()`（零矩形），平台提供自有装饰或无标题栏 |

用途：当在自定义标题栏中绘制内容（标题、搜索框、工具栏）时，用此矩形避开装饰按钮区域。

---

## 窗口事件结构体

### `WindowGeomChangeEvent`
- `window_id: WindowId` — 窗口标识
- `old_geom: WindowGeom` — 变更前的几何
- `new_geom: WindowGeom` — 变更后的几何

当窗口的位置、大小、DPI、全屏状态、安全区等发生任何变化时发送。

### `WindowMovedEvent`
- `old_pos: Vec2d` — 移动前的位置
- `new_pos: Vec2d` — 移动后的位置

### `WindowCloseRequestedEvent`
- `window_id: WindowId`
- `accept_close: Rc<Cell<bool>>` — 是否允许关闭的标记

使用 `Rc<Cell<bool>>` 使多个处理器可以协作决定是否允许窗口关闭。

### `WindowClosedEvent`
- `window_id: WindowId` — 已关闭的窗口标识

### `PopupDismissReason` 枚举

弹窗关闭的原因分类：
- `FocusLost` — 弹窗失去焦点
- `OutsideClick` — 在弹窗外点击
- `Escape` — 按下 Esc 键
- `Compositor` — 合成器主动关闭（Wayland 的 PopupDone）
- `ParentClosed` — 父窗口关闭

### `PopupDismissedEvent`
```rust
pub struct PopupDismissedEvent {
    pub window_id: WindowId,
    pub reason: PopupDismissReason,
}
```
通知一个弹窗应该被关闭。**应用必须主动调用 `WindowHandle::close()` 来实际关闭弹窗**，框架不会自动关闭。

### `WindowDragQueryResponse` 枚举

窗口拖拽查询的响应类型：
- `NoAnswer` — 不提供明确回答
- `Client` — 客户端内容区
- `Caption` — 标题栏区域（可拖动）
- `SysMenu` — 系统菜单区域（仅 Windows）

### `WindowDragQueryEvent`
```rust
pub struct WindowDragQueryEvent {
    pub window_id: WindowId,
    pub abs: Vec2d,           // 查询位置
    pub response: Rc<Cell<WindowDragQueryResponse>>,  // 响应结果
}
```
平台询问窗口指定位置是否可拖动。widget 可以在命中测试时设置 `response` 值，告知平台该位置是否属于标题栏（可拖动区域）。
