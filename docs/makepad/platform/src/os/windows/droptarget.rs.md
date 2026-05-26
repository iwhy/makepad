# droptarget.rs — COM IDropTarget 接口实现用于接收拖放

**文件路径**: platform/src/os/windows/droptarget.rs (237行)
**核心用途**: 实现 Windows COM `IDropTarget` 接口，接收外部文件拖放，通过 `WM_USER` 消息转发到 Makepad 窗口。

## 枚举

### `DropTargetMessage`
```rust
pub enum DropTargetMessage {
    Enter(MODIFIERKEYS_FLAGS, POINTL, DROPEFFECT, DragItem), // 拖入
    Leave,                                                    // 离开
    Over(MODIFIERKEYS_FLAGS, POINTL, DROPEFFECT, DragItem),  // 悬停
    Drop(MODIFIERKEYS_FLAGS, POINTL, DROPEFFECT, DragItem),  // 放下
}
```

## 常量

- `WM_DROPTARGET: u32 = WM_USER + 0` — 用于转发拖放事件的自定义窗口消息

## 结构体

### `DropTarget`
```rust
pub(crate) struct DropTarget {
    pub drag_item: RefCell<Option<DragItem>>, // 缓存的拖放项（Windows 只对 Enter/Drop 提供数据）
    pub hwnd: HWND,                           // 目标窗口句柄
}
```
通过 `implement_com!` 宏注册为 COM 对象，实现 `IDropTarget`。

## 实现细节

### 辅助函数 `create_dragitem_from_idataobject`

1. 调用 `data_object.EnumFormatEtc(DATADIR_GET)` 获取格式枚举器
2. 枚举所有支持的 `FORMATETC` 格式，查找 `CF_HDROP`
3. 调用 `data_object.GetData()` 获取 `STGMEDIUM`
4. 调用 `convert_medium_to_dragitem()` 解析为 `DragItem`

### IDropTarget_Impl for DropTarget_Impl

| COM 方法 | 行为 |
|----------|------|
| `DragEnter` | 从 IDataObject 创建 DragItem，缓存到 `drag_item`，通过 `SendMessageW(WM_DROPTARGET)` 发送 `Enter` 消息 |
| `DragLeave` | 清除 `drag_item` 缓存，发送 `Leave` 消息 |
| `DragOver` | 使用缓存的 `drag_item` 发送 `Over` 消息（Windows 原生不提供 Over 时的数据） |
| `Drop` | 重新从 IDataObject 获取 DragItem，清除缓存，发送 `Drop` 消息 |

### 消息传递机制

使用 `WM_USER + 0` (WM_DROPTARGET) 将 COM 回调转换为窗口消息：
- `WPARAM` = 0
- `LPARAM` = `Box::into_raw(Box::new(DropTargetMessage))` 作为 `isize`
- 接收方负责 `Box::from_raw()` 取回并释放

## 平台集成

- 通过 `win32_window.rs` 注册 `IDropTarget` 到窗口（调用 `RegisterDragDrop`）
- 在 `win32_app.rs` 的窗口消息循环中处理 `WM_DROPTARGET`
- 转换为 Makepad 的 `DragEvent` / `DropEvent` 传递给 Cx 事件处理
