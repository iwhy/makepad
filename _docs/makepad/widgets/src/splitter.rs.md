# splitter.rs — Splitter 可拖拽分割器

## 文件概述

`Splitter` 是一个可拖拽的分割条组件，用于在水平或垂直方向上分割两个面板区域，允许用户通过拖拽调整两侧面板的尺寸比例。

---

## `Splitter` 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct Splitter {
    #[source] source: ScriptObjectRef,
    #[walk] walk: Walk,
    #[layout] layout: Layout,
    #[redraw] #[live] draw_bg: DrawQuad,
    #[live] animator: Animator,
    #[live] orientation: Axis,        // 分割方向
    #[live] min_size: f64,            // 任一方向面板最小尺寸
    #[live] grab_size: f64,           // 拖拽热区尺寸
    #[rust] is_dragging: bool,
    #[rust] drag_start_pos: DVec2,
    #[rust] drag_start_ratio: f64,    // 拖拽开始时的比例
}
```

---

## 核心逻辑

### 方向与布局

Splitter 支持两种方向：

- `Axis::X`：水平分割（左右面板），Splitter 表现为垂直条，可水平拖拽。
- `Axis::Y`：垂直分割（上下面板），Splitter 表现为水平条，可垂直拖拽。

布局中，Splitter 占据固定像素宽度（`grab_size`）：

```
// 水平分割 (Axis::X)
┌─────────┬──┬─────────┐
│         │  │         │
│  Panel1 │▊▊│ Panel2  │
│         │  │         │
└─────────┴──┴─────────┘
     ↑          ↑
  可变宽度   可变宽度
```

### 拖拽处理

```
MouseDown → 记录 drag_start_pos 和当前比例
    ↓
MouseMove → 计算鼠标位置变化 → 更新分割比例
    ↓
MouseUp → 锁定比例
```

1. MouseDown 在 grab 区域内时，设置 `is_dragging = true`，记录初始鼠标位置 `drag_start_pos` 和当前分割比例 `drag_start_ratio`。
2. MouseMove 时计算增量：
   ```rust
   fn update_ratio(&mut self, mouse_pos: DVec2) {
       let delta = if self.orientation == Axis::X {
           mouse_pos.x - self.drag_start_pos.x
       } else {
           mouse_pos.y - self.drag_start_pos.y
       };
       let total_size = self.parent_size - self.grab_size;
       let new_ratio = self.drag_start_ratio + delta / total_size;
       self.ratio = new_ratio.clamp(self.min_ratio, 1.0 - self.min_ratio);
   }
   ```
3. 比例约束在 `[min_ratio, 1.0 - min_ratio]` 之间，`min_ratio` 由 `min_size / total_size` 计算。

### 鼠标光标变化

拖拽过程中改变鼠标光标样式：

- 水平分割：`cursor: MouseCursor.ColResize`（↔）
- 垂直分割：`cursor: MouseCursor.RowResize`（↕）

在 MouseMove 中检测 hit test，当鼠标悬停在 grab 区域上时设置光标。

---

## 与 Dock 系统的关系

`Splitter` 是独立的基础分割器组件。在 Dock 系统中，`DockSplitter` 在其基础上增加了标签页分组等功能。Dock 系统使用多个 Splitter 实例来构建复杂的可拖拽面板布局。

---

## 动画集成

- `animator` 用于拖拽过程中的平滑过渡效果（如 Snap 动画）。
- 但 Splitter 的拖拽是实时的，主要依赖直接比例计算而非动画插值。
- 动画主要用于悬停/按下状态的颜色变化和光标过渡。
