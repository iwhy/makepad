# widget_tree.rs — WidgetTree 持久图结构、查询索引与 CxWidgetExt

## 文件概述

定义了 `WidgetTree` 结构体，它是 Makepad UI 框架中 Widget 的持久化图结构。与传统的即时创建/销毁树不同，`WidgetTree` 在帧间持久存在，维护了 `LiveId` → `WidgetNode` 的映射关系和滑动索引。

---

## `WidgetTree` 结构体

```rust
pub struct WidgetTree {
    area_to_widget: Vec<AreaToWidget>,  // Area → Widget 查找表
    widget_to_areas: Vec<WidgetToArea>, // Widget → Area 反向映射
    abi_area: AbiArea,                  // ABI 兼容的 Area 类型
    abi_widget: AbiWidget,              // ABI 兼容的 Widget 索引
    area_index: usize,                  // Area 索引计数器
}
```

核心功能：

1. **Area→Widget 映射**：给定一个鼠标点击的 Area，快速查找对应的 Widget 引用。
2. **Widget→Area 映射**：从 Widget 索引找到其注册的所有 Area。
3. **滑动索引**：在 `begin`/`end` 之间，`area_index` 持续递增，超出历史长度的索引自动创建新条目。
4. **ABI 兼容性**：`AbiArea` 和 `AbiWidget` 是跨 DLL/动态链接边界的轻量句柄。

### 生命周期管理

```rust
pub fn begin(&mut self)     // 每帧开始，重置 area_index
pub fn end(&mut self)       // 每帧结束，清理过期条目
```

- `begin()` 在帧开始时调用，重置 `area_index` 并准备映射更新。
- `end()` 在帧结束时调用，通过 `Vec::drain(0..old_len)` 移除上一帧未更新的旧条目，保持树与当前帧同步。

### `add_area` 与关联

```rust
pub fn add_area(&mut self, area: &Area) -> AreaIndex
pub fn add_widget(&mut self, index: AreaIndex, widget: AbiWidget, id: Option<LiveId>)
```

- `add_area(area)`：注册一个 Area，返回递增的 `AreaIndex`。
- `add_widget(index, widget, id)`：将 Widget 引用与 AreaIndex 关联，可选的 `LiveId` 用于具名查找。

### 查询接口

```rust
pub fn find_from_point(&self, point: DVec2) -> Option<WidgetTreeQuery>
```

从屏幕坐标点命中测试：
1. 逆序遍历 `area_to_widget`（后绘制的在上层）。
2. 对每个 Area 调用 `point_inside(point)` 进行矩形碰撞检测。
3. 返回命中的 `WidgetTreeQuery`，包含 Widget 引用和 Area。

### `WidgetTreeQuery`

```rust
pub struct WidgetTreeQuery {
    pub widget: Option<AbiWidget>,  // 命中的 Widget 引用
    pub area: Area,                 // 命中的点击区域
}
```

---

## `AreaToWidget` 和 `WidgetToArea`

- `AreaToWidget { area: Area, widget: AbiWidget, id: Option<LiveId> }` — 正向映射，用于命中检测。
- `WidgetToArea { area: Area, index: usize }` — 反向映射，用于查找 Widget 的所有 Area。

---

## `CxWidgetTreeExt`

为 `Cx` 提供的扩展：

```rust
pub trait CxWidgetTreeExt {
    fn begin_widget_tree(&mut self, widget_tree: &mut WidgetTree);
    fn end_widget_tree(&mut self, widget_tree: &mut WidgetTree);
    fn add_widget_area(&mut self, widget_tree: &mut WidgetTree, id: Option<LiveId>);
}
```

- `begin_widget_tree`：在帧开始时调用 `widget_tree.begin()`。
- `end_widget_tree`：在帧结束时调用 `widget_tree.end()`。
- `add_widget_area`：将当前 turtle 绘制的区域注册到 widget_tree 中。这是最常用的方法——每个 Widget 在绘制完成时调用它，将自身 Area 与 Widget 索引关联。

---

## `Area` 与命中测试

`Area` 是 Makepad 基础库中的正矩形点击区域。`WidgetTree` 通过逆序遍历已注册的 Area 进行碰撞检测：

```rust
while let Some(atw) = self.area_to_widget.get(index) {
    if atw.area.point_inside(point) {
        return Some(WidgetTreeQuery { ... });
    }
}
```

- 逆序确保叠放次序正确：Widget 在数组中的位置由帧内的注册顺序决定，后注册的在上层。
- `point_inside` 使用非包容的下边界检查，符合图形学惯例。
