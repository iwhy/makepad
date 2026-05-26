# dropfiles.rs — DragItem 与 Win32 DROPFILES 结构转换

**文件路径**: platform/src/os/windows/dropfiles.rs (176行)
**核心用途**: 实现 Makepad `DragItem` 与 Windows `DROPFILES` / `HGLOBAL` 内存结构之间的二进制序列化和反序列化。

## 关键函数

### `convert_medium_to_dragitem(medium: STGMEDIUM) -> Option<DragItem>`

从 COM `STGMEDIUM` 中提取 `DragItem`。

**接收流程**:
1. 调用 `GlobalSize` 获取 HGLOBAL 大小，调用 `GlobalLock` 获取原始指针
2. 将前 7 个 `u32` 作为 `DROPFILES` 头部解析：
   - `u32_slice[0]` = `names_offset` (文件名偏移量)
   - `u32_slice[4]` = `has_wide_strings` (宽字符标志)
3. 校验：必须使用宽字符串 (`has_wide_strings != 0`)；`names_offset` 必须为 20 或 28
4. 根据偏移量读取：
   - **偏移量 20**：标准外部 DROPFILES (来自文件管理器)
   - **偏移量 28**：内部格式，从 `u32_slice[5]` 和 `[6]` 提取 64 位 `LiveId`
5. 解码 UTF-16 空终止字符串序列为 `Vec<String>`
6. 当前限制：只支持单文件拖放，多文件返回 `None`
7. 返回 `DragItem::FilePath { path, internal_id }`

### `create_hglobal_for_dragitem(drag_item: &DragItem) -> Option<HGLOBAL>`

从 `DragItem` 创建 HGLOBAL 存储。

**发送流程**:
1. 仅支持 `DragItem::FilePath` 变体
2. 将文件路径编码为 UTF-16 并添加双空终止 (文件名 + 列表终止)
3. 分配 HGLOBAL: 28 字节头部 + 文件名数据
4. 写入 `DROPFILES` 结构：
   - `u32_slice[0]` = 28 (偏移量)
   - `u32_slice[4]` = 1 (宽字符标志)
5. 如果存在 `internal_id`，写入 `u32_slice[5]` (低32位) 和 `u32_slice[6]` (高32位)
6. 使用 `GlobalUnlock` 释放指针并返回 HGLOBAL

## DROPFILES 内存布局

```
偏移量 0:  u32  pFiles (到文件名的偏移量)
偏移量 4:  POINT pt (文件拖放点, 未使用)
偏移量 16: BOOL fWide (宽字符标志)
偏移量 20: u32 (内部 ID 低32位, 仅内部格式)
偏移量 24: u32 (内部 ID 高32位, 仅内部格式)
偏移量 20/28: UTF-16 文件名列表 (空终止, 双空终止结尾)
```

## 平台集成

- 与 `droptarget.rs` 配合解析传入的拖放数据
- 与 `dataobject.rs` 配合序列化传出的拖放数据
- 使用 `win32::System::Ole::ReleaseStgMedium` 释放 COM 介质
