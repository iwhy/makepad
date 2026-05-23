# `math_usize.rs` 源码解读

**路径:** `libs/math/src/math_usize.rs`
**行数:** 81
**核心职责:** 无符号整数精度的点、尺寸、矩形类型，用于像素精确的布局计算

---

## 类型定义

### `RectUsize`
- **[origin]**: `PointUsize` — 左上角原点
- **[size]**: `SizeUsize` — 宽高
- 整数精度的轴对齐矩形

### `PointUsize`
- **[x]**: `usize` — 横坐标
- **[y]**: `usize` — 纵坐标

### `SizeUsize`
- **[width]**: `usize` — 宽度
- **[height]**: `usize` — 高度
- 额外派生 `Eq + Hash + PartialEq`，可用作 HashMap 键

---

## impl 块详解

### `impl RectUsize`

#### `pub const fn new(origin: PointUsize, size: SizeUsize) -> Self`
- 常量构造函数

#### `pub const fn min(self) -> PointUsize`
- 返回左上角（即 origin）

#### `pub fn max(self) -> PointUsize`
- 计算右下角：`origin + size`（通过 Add<SizeUsize> 实现）

#### `pub fn union(self, other: Self) -> Self`
- 计算两个矩形的并集包围盒
- 取两个矩形的 min 和 max 作为新矩形边界
- `max - min` 得到新的尺寸

---

### `impl PointUsize`

#### `pub const fn new(x: usize, y: usize) -> Self`
- 常量构造函数

#### `pub fn min(self, other: Self) -> Self`
- 逐分量取最小值

#### `pub fn max(self, other: Self) -> Self`
- 逐分量取最大值

---

## 运算符重载

### `impl Add<SizeUsize> for PointUsize`
- **点加尺寸**：`x + width`，`y + height`，返回新 PointUsize

### `impl Sub for PointUsize`
- **点减点**：`x1 - x2`，`y1 - y2`，返回 SizeUsize

---

### `impl SizeUsize`

#### `pub const fn new(width: usize, height: usize) -> Self`
- 常量构造函数
