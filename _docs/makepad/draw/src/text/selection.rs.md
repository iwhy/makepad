# `draw/src/text/selection.rs` — 文本选择与光标模型

## 概述

这是 Makepad 文本子系统中最小的模块（36 行），定义了三个核心数据结构：`Selection`（选择范围）、`Cursor`（光标位置）、`CursorPosition`（光标的视觉位置）。这些类型被 `LaidoutText` 用于计算选择高亮矩形和光标定位。

## 核心类型

### `Cursor`（第 24-29 行）
**逻辑光标**。包含：
- `index: usize` — 在文本中的字节偏移
- `prefer_next_row: bool` — 确定光标在行边界时的行选择偏好。例如当光标在行尾时，`prefer_next_row = true` 使光标出现在下一行开头而非当前行尾

实现了 `Eq + Ord + PartialOrd`（按 index 比较），用于确定选择和导航的顺序。

### `CursorPosition`（第 31-35 行）
**视觉光标位置**。包含：
- `row_index: usize` — 在已布局行中的索引
- `x_in_lpxs: f32` — 在该行中的水平位置（逻辑像素）

### `Selection`（第 1-22 行）
**文本选择范围**。包含：
- `cursor: Cursor` — 当前光标位置
- `anchor: Cursor` — 锚点位置（选择起始或结束端）

#### `Selection::start() -> Cursor`
返回选择范围的起点（`cursor.min(anchor)`）。基于 `Cursor` 的 `Ord` 实现（按 index 比较）。

#### `Selection::end() -> Cursor`
返回选择范围的终点（`cursor.max(anchor)`）。

#### `Selection::index_eq(other) -> bool`
比较两个 Selection 的 cursor 和 anchor 的 index 是否分别相等（忽略 `prefer_next_row` 差异）。用于判断选择是否发生变化。

## 与其他模块的关系

这些结构体在 `layouter.rs` 中被 `LaidoutText` 的方法广泛使用：
- `cursor_to_position()` 将 `Cursor` 转换为 `CursorPosition`（行索引 + x 坐标）
- `point_in_lpxs_to_cursor()` 将屏幕坐标（点击点）转换为 `Cursor`
- `position_to_cursor()` 将 `CursorPosition` 转换回 `Cursor`
- `selection_rects()` 基于 `Selection` 计算高亮矩形列表

## 序列化支持

所有三个类型都带有 `#[cfg_attr(feature = "serde", derive(...))]` 条件编译属性，当启用 `serde` feature 时自动实现 `Serialize` 和 `Deserialize`。
