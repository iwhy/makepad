# `draw/src/cx_2d.rs` — Cx2d：2D 绘图上下文

## 类型定义

```rust
pub struct Cx2d<'a, 'b> {
    pub cx: &'b mut CxDraw<'a>,
    pub(crate) overlay_id: Option<DrawListId>,
    pub(crate) overlay_pass_id: Option<DrawPassId>,
    pub(crate) overlay_draw_depth: usize,
    pub(crate) turtles: Vec<Turtle>,
    pub(crate) finished_rows: Vec<usize>,
    pub(crate) finished_walks: Vec<FinishedWalk>,
    pub(crate) turtle_clips: Vec<(Vec2d, Vec2d)>,
    pub(crate) align_list: Vec<AlignEntry>,
    pub(crate) draw_call_parent_stack: Vec<u64>,
    pub(crate) draw_call_parent_next: u64,
}
```

### 职责

Cx2d 是 **2D 绘制的最上层上下文**。它包装了 `CxDraw`，并增加了以下管理职责：

- **Turtle 布局引擎**：管理 Turtle 堆栈（`turtles`）、已完成行/步（`finished_rows`/`finished_walks`）和 clip 区域
- **对齐列表**：`align_list` 存储布局对齐条目，供后续对齐计算
- **覆盖层绘制**：通过 `overlay_id`、`overlay_pass_id`、`overlay_draw_depth` 追踪当前是否处于覆盖层绘制中
- **DrawCall 分组**：通过 `draw_call_parent_stack` 实现背景/内容/前景的分层分组

### `Deref`/`DerefMut` 到 `CxDraw`

```rust
impl<'a, 'b> Deref for Cx2d<'a, 'b> {
    type Target = CxDraw<'a>;
    fn deref(&self) -> &Self::Target { self.cx }
}
```

**重要设计**：Cx2d 自动解引用到 CxDraw，这意味着 `&mut Cx2d` 可以直接调用 CxDraw 的所有方法，无需额外的间接层。

## `new(cx)` — 构造函数

```rust
pub fn new(cx: &'b mut CxDraw<'a>) -> Self
```

### 实现逻辑

1. 分配初始容量为 256 的 `draw_call_parent_stack`
2. 推送根 scope ID = `1` 作为初始分组父节点
3. 将所有容器（`turtles`、`finished_rows`、`finished_walks`、`turtle_clips`、`align_list`）预分配合理初始容量，减少运行时扩容
4. `draw_call_parent_next` 从 `2` 开始，确保 ID 唯一递增

### 数据结构容量设定

| 字段 | 初始容量 | 用途 |
|------|---------|------|
| `draw_call_parent_stack` | 256 | 分组栈深度 |
| `turtle_clips` | 1024 | clip 区域栈 |
| `finished_rows` | 1024 | 已完成行数 |
| `finished_walks` | 1024 | 已完成布局步数 |
| `turtles` | 64 | Turtle 堆栈 |
| `align_list` | 4096 | 对齐条目 |

## `is_drawing_overlay()` — 是否正在绘制覆盖层

```rust
pub fn is_drawing_overlay(&self) -> bool {
    self.overlay_draw_depth > 0
}
```

当 `overlay_draw_depth > 0` 时，表示当前绘制调用来自覆盖层 UI。

## DrawCall 分组机制

这是 Cx2d 中较核心但也较简短的机制。它通过 `draw_call_parent_stack`（Vec\<u64\>）为绘制调用创建一个层级分组 ID 系统。

### `push_draw_call_parent()` / `pop_draw_call_parent()`

```rust
pub fn push_draw_call_parent(&mut self) {
    let id = self.draw_call_parent_next;
    self.draw_call_parent_next = self.draw_call_parent_next.wrapping_add(1).max(2);
    self.draw_call_parent_stack.push(id);
}

pub fn pop_draw_call_parent(&mut self) {
    if self.draw_call_parent_stack.len() > 1 {
        self.draw_call_parent_stack.pop();
    }
}
```

#### 实现说明

- `push_draw_call_parent()`：分配一个新的唯一 64 位 ID 并将其推入栈中，用于标记新的分组边界。
  - `.wrapping_add(1).max(2)` 确保 ID 不会回绕到 0 或 1（0 未使用，1 是根分组的保留 ID）。
- `pop_draw_call_parent()`：从栈顶弹出当前分组 ID，回到父级分组。栈底 root ID (1) 不被弹出，保证至少有一个父节点。

### `pack_draw_call_group(base, lane)`

```rust
fn pack_draw_call_group(base: u64, lane: u8) -> LiveId {
    LiveId((base << 8) | lane as u64)
}
```

将基础 ID（父节点 ID）与子车道号（0=背景，1=内容）编码为一个 `LiveId`。

### `draw_call_group_parent()` — 获取当前父分组 ID

```rust
pub fn draw_call_group_parent(&self) -> LiveId {
    LiveId(self.draw_call_parent_stack[len - 2])
}
```

取栈中倒数第二个元素（当前分组的父级 ID）。

### `draw_call_group_current()` — 获取当前分组 ID

```rust
pub fn draw_call_group_current(&self) -> LiveId {
    LiveId(*self.draw_call_parent_stack.last().unwrap_or(&1))
}
```

取栈顶元素作为当前分组 ID，栈为空时默认返回 root (1)。

### `draw_call_group_background()` — 当前父分组的背景车道

```rust
pub fn draw_call_group_background(&self) -> LiveId {
    Self::pack_draw_call_group(base, 0)
}
```

取父节点 ID 加上 `lane=0`，表示该分组的背景绘制调用。

### `draw_call_group_content()` — 当前父分组的内容车道

```rust
pub fn draw_call_group_content(&self) -> LiveId {
    if len >= 2 {
        Self::pack_draw_call_group(base, 1)
    } else {
        Self::pack_draw_call_group(self.draw_call_parent_stack[0], 0)
    }
}
```

#### 实现逻辑

1. 如果当前不是顶级 scope（`len >= 2`），取父级 ID 加 `lane=1`，让所有子 scope 的内容共享同一个 lane
2. 如果是顶级 scope，则使用 `lane=0`（与背景相同），因为顶级不需要区分背景/内容

这使得背景绘制调用（lane 0）和内容绘制调用（lane 1）可以被 GPU 批处理引擎合并处理，减少 draw call 数量。

## 脏矩形检测

### `will_redraw(draw_list_2d, walk)`

```rust
pub fn will_redraw(&self, draw_list_2d: &mut DrawList2d, walk: Walk) -> bool
```

#### 实现逻辑

1. 通过 `peek_walk_turtle(walk)` 计算当前 Turtle 在给定 `walk` 下的预期矩形位置
2. 与 `draw_list_2d.dirty_check_rect` 比较：
   - 如果位置/大小发生变化 → 设置新的 dirty rect 并返回 `true`（需要重绘）
   - 如果未变化 → 委托 `draw_event.draw_list_will_redraw()` 检查该 draw list 的增量重绘标志

这是 **增量重绘优化** 的核心：当 widget 的新旧位置/尺寸一致时，无需清除并重新构建 draw call，可以直接复用上一帧的绘制数据。

### `will_redraw_check_axis(draw_list_2d, size, axis)`

```rust
pub fn will_redraw_check_axis(
    &self,
    draw_list_2d: &mut DrawList2d,
    size: f64,
    axis: Vec2Index,
) -> bool
```

#### 实现逻辑

与 `will_redraw` 类似，但只检查单一轴（X 或 Y）的尺寸变化。用于只在一个方向滚动的列表等高瘦场景，减少脏矩形检测的粒度。

1. 比较指定轴的尺寸与 `dirty_check_rect.size.index(axis)`：
   - 变化 → 更新该轴尺寸，返回 `true`
   - 未变 → 委托 `draw_event.draw_list_will_redraw()`
