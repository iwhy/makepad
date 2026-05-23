# `turtle.rs` — Makepad 海龟布局引擎

**文件路径**: `draw/src/turtle.rs` (2747 行)  
**核心作用**: 实现 Makepad 特有的"海龟图形"布局引擎，类似于 CSS Flexbox 但基于光标推进模型（turtle cursor）。所有 UI 控件的空间分配与排列都由此文件驱动。

---

## 一、核心类型体系

### 1. `Walk`（行走规格，第 49–69 行）

`Walk` 描述一次布局行走的期望尺寸与外边距，是布局请求的"输入参数"。

```rust
pub struct Walk {
    pub abs_pos: Option<Vec2d>,   // 绝对位置（可选），设置后忽略流式布局
    pub margin: Inset,            // 外边距
    pub width: Size,              // 期望宽度
    pub height: Size,             // 期望高度
    pub metrics: Metrics,         // 文字度量信息（descender/line_gap/line_scale）
}
```

**构造方法**:
- `Walk::new(width, height)` — 指定宽高的 Walk
- `Walk::empty()` — 固定 0×0
- `Walk::fill()` — 宽高均为 `Size::fill()`，填满可用空间
- `Walk::fixed(w, h)` — 固定像素尺寸
- `Walk::fit()` — 自适应内容尺寸
- `Walk::fill_fit()` — 宽填满、高自适应
- `Walk::abs_rect(rect)` — 绝对位置矩形

**链式方法**: `with_margin()`, `with_abs_pos()`, `with_margin_all()`, `with_add_padding()`

---

### 2. `Size`（尺寸规格，第 204–301 行）

`Size` 是 `Walk` 的宽/高值的枚举，控制布局引擎如何解释尺寸请求：

| 变体 | 含义 | 行为 |
|------|------|------|
| `Fill { weight, min, max }` | 填满可用空间 | 根据剩余空间分配，支持权重、最小/最大约束 |
| `Fixed(f64)` | 固定尺寸 | 直接使用给定值 |
| `Fit { min, max }` | 自适应内容 | 由子元素撑大，支持最小/最大约束 |

**`Size::Fill` 的权重系统**（第 1144–1200 行）：  
当同一容器内有多个 `Fill` 元素时，剩余空间按照 `weight` 比例分配。通过 `deferred_fills` 列表延迟计算，支持 `min`/`max` 约束。算法在 `resolve_fill()` 中实现：

1. 计算从当前索引开始的未分配空间长度 `unresolved_length`
2. 计算从当前索引开始的总权重 `total_deferred_weight`
3. 分配 `length = unresolved_length * weight / total_deferred_weight`
4. 应用 `min`/`max` 约束

**`Size::Fit` 的边界系统**（第 303–361 行）：  
`FitBound` 可以是绝对数值 `Abs(f64)` 或相对于父容器尺寸的 `Rel { base, factor }`。`Base` 枚举有 `Full`（父容器完整尺寸）和 `Unused`（父容器未用尺寸）两个变体。在 `compute_final_size()`（第 1689–1743 行）中评估。

**`ScriptHook` 支持**（第 277–301 行）：  
`Size` 实现了 `ScriptHook`，允许在 DSL 中直接用数字字面量（如 `width: 200.0`）作为 `Size::Fixed`。

---

### 3. `Layout`（布局规格，第 364–482 行）

`Layout` 控制海龟内部子元素的排列方式，相当于 CSS Flexbox 的 `display: flex` + 方向 + 间距：

```rust
pub struct Layout {
    pub scroll: Vec2d,        // 滚动偏移
    pub clip_x: bool,         // 是否水平裁剪（默认 true）
    pub clip_y: bool,         // 是否垂直裁剪（默认 true）
    pub flow: Flow,           // 布局流向
    pub spacing: f64,         // 元素间距
    pub wrap_spacing: f64,    // 折行间距（仅 Flow::Right { wrap: true }）
    pub padding: Inset,       // 内边距
    pub align: Align,         // 对齐方式
}
```

**构造方法**:
- `Layout::flow_right()` — 从左到右
- `Layout::flow_right_wrap()` — 从左到右，自动折行
- `Layout::flow_down()` — 从上到下
- `Layout::flow_overlay()` — 叠放

**链式方法**: `with_padding()`, `with_scroll()`, `with_align()`, `with_clip()`, `with_padding_all()`

---

### 4. `Flow`（流向，第 503–545 行）

```rust
pub enum Flow {
    Right { row_align: RowAlign, wrap: bool },  // 左→右，可选折行
    Down,                                        // 上→下
    Overlay,                                     // 叠放
}
```

默认值为 `Flow::Right { row_align: RowAlign::Top, wrap: false }`。

---

### 5. `Align`（对齐，第 485–500 行）

`Align` 在容器有剩余空间时调整子元素位置，分值 `x`（水平）和 `y`（垂直）：

```rust
pub struct Align {
    pub x: f64,  // 0.0=左/上, 0.5=居中, 1.0=右/下
    pub y: f64,
}
```

预定义常量（第 18–21 行）：
- `TopLeft = Align { x: 0.0, y: 0.0 }`
- `Center = Align { x: 0.5, y: 0.5 }`
- `HCenter = Align { x: 0.5, y: 0.0 }`
- `VCenter = Align { x: 0.0, y: 0.5 }`

---

### 6. `Inset`（插入间距）

`Inset` 是 `margin` 和 `padding` 使用的四边间距类型，定义在平台 crate 中，包含 `left`, `right`, `top`, `bottom` 四个 `f64` 字段。提供了 `left_top()`, `width()`, `height()` 等便捷方法。

---

### 7. `Metrics`（文字度量，第 171–199 行）

```rust
pub struct Metrics {
    pub descender: f64,    // 下行部分高度
    pub line_gap: f64,     // 行间距
    pub line_scale: f64,   // 行缩放比例（默认 1.0）
}
```

用于 `RowAlign::Bottom` 基准线对齐模式下的行间距计算。

---

### 8. `RowAlign`（行对齐，第 553–569 行）

仅在 `Flow::Right { wrap: true }` 时生效：

| 变体 | 行为 |
|------|------|
| `Top` | 顶部对齐（默认，无后处理） |
| `Bottom` | 底部基准线对齐，使用 `Metrics.descender` 计算偏移 |
| `Center` | 垂直居中对齐，适合内联块元素与文本混合的场景 |

---

## 二、海龟（Turtle）数据结构（第 593–612 行）

```rust
pub struct Turtle {
    walk: Walk,                    // 创建时的 Walk
    layout: Layout,                // 布局参数
    width: f64,                    // 实际宽度（可能 NaN 表示未知）
    height: f64,                   // 实际高度（可能 NaN 表示未知）
    used_width: f64,               // 已使用宽度
    used_height: f64,              // 已使用高度
    prev_row_metrics: Metrics,     // 上一行度量
    current_row_metrics: Metrics,  // 当前行度量
    wrap_spacing: f64,             // 折行间距
    align_start: usize,            // align_list 中本海龟的起始索引
    finished_rows_start: usize,    // finished_rows 中本海龟的起始索引
    finished_walks_start: usize,   // finished_walks 中本海龟的起始索引
    deferred_fills: Vec<DeferredFill>,  // 延迟填充元素列表
    resolved_fills: Vec<f64>,           // 已解析的填充尺寸
    pos: Vec2d,                    // 当前光标位置
    origin: Vec2d,                 // 海龟矩形原点
    guard: Area,                   // 保护区域（用于 begin/end 配对校验）
}
```

海龟的内部矩形层次为：
```
+---------------------+
| Padding Inset       |
| +-----------------+ |
| | Margin Inset    | |
| | +-------------+ | |
| | | Content     | | |
| | +-------------+ | |
| +-----------------+ |
+---------------------+
```

- 内部矩形（inner rect）= content
- 矩形（rect）= content + padding
- 外部矩形（outer rect）= content + padding + margin

---

## 三、核心布局算法

### 3.1 `begin_root_turtle()`（第 1331–1359 行）

初始化根海龟，将 `AlignEntry::BeginClip` 压入 `align_list`，创建带有完整尺寸的 `Turtle`。

- **参数**: `size`（视口尺寸）、`layout`（布局参数）
- **行为**: 设置 `origin = (0,0)`，`pos = (padding.left, padding.top)`，宽度/高度直接设为 `size`，初始化所有行/行走计数器和度量。

### 3.2 `begin_turtle()` / `begin_turtle_with_guard()`（第 1391–1460 行）

创建嵌套海龟。这是布局递归的入口——每个 UI 控件调用此方法来创建子布局上下文：

1. 读取父海龟的 `pos` 和 `next_walk_size()` 计算子海龟的尺寸
2. 根据 `walk.abs_pos` 决定绝对定位还是流式定位
3. 应用 `walk.margin` 得到 `origin`
4. 根据 `layout.clip_x/clip_y` 创建裁剪区域
5. 应用 `layout.scroll` 滚动偏移
6. 将 `AlignEntry::BeginClip` 插入 `align_list`
7. 新建 `Turtle` 压入 `turtles` 栈

**关于 `guard` 参数**：用于 begin/end 配对校验，确保不会错配海龟边界。

### 3.3 `end_turtle()` / `end_turtle_with_guard()`（第 1463–1687 行）

结束当前海龟，执行对齐和尺寸最终化：

1. **`finish_row()`**（第 2171–2193 行）— 结束当前行，对行内元素应用 `RowAlign` 对齐
2. **`compute_final_size()`**（第 1689–1743 行）— 计算 `Fit` 尺寸的最终值，应用 `min`/`max` 约束
3. **对齐处理**（第 1486–1662 行）：
   - **`Flow::Right`（无折行）**：水平对齐整体应用，垂直对齐逐个应用；有 deferred fills 时按权重分配剩余宽度
   - **`Flow::Right`（有折行）**：对每行单独应用水平对齐
   - **`Flow::Down`**：水平对齐逐个应用，垂直对齐整体应用
   - **`Flow::Overlay`**：水平和垂直均逐个应用
4. 压入 `EndClip`，截断 `finished_walks`/`finished_rows` 至海龟的起始位置
5. 弹出海龟，将自身作为 `walk_turtle_internal()` 的输入回写给父海龟

### 3.4 `walk_turtle()` / `walk_turtle_internal()`（第 1846–1929 行）

**这是布局引擎的核心方法**，在海龟内部行走一步，分配一个矩形空间：

1. 调用 `turtle.next_walk_size()` 计算实际尺寸
2. **绝对定位分支**（`walk.abs_pos.is_some()`）：
   - 移动到指定位置
   - 根据流向调用 `allocate_height/width/size()`
   - 恢复旧位置
3. **流式定位分支**（第 1878–1928 行）：
   - 计算 `spacing` 偏移（不是第一个元素时应用 `layout.spacing`）
   - 根据流向分配位置：
     - **`Flow::Right { wrap: true }`**：如果超出当前行剩余宽度，调用 `wrap_turtle()` 自动折行
     - **`Flow::Right`**：向右移动，分配宽度
     - **`Flow::Down`**：向下移动，分配高度
     - **`Flow::Overlay`**：在原点叠加
   - 更新 `current_row_metrics`（取本行所有行走的 `Metrics` 最大值，用于 `RowAlign::Bottom`）
   - 记录 `deferred_before_count`（用于后续的 fill 解析）

### 3.5 `next_walk_width()` / `next_walk_height()`（第 1036–1098 行）

根据 `Size` 类型和流向计算行走的实际尺寸：

**宽度计算**：
- `Size::Fill`：`Flow::Right` 且无折行时 = 未用内宽；有折行时 = 当前行未用内宽；`Flow::Down/Overlay` 时 = 有效内宽；然后应用 `min/max` 约束，减去 margin 宽度
- `Size::Fixed`：直接返回给定值（最小 0）
- `Size::Fit`：返回 NaN（尺寸未知，延迟到 `compute_final_size()` 确定）

**高度计算**：
- `Size::Fill`：`Flow::Right/Overlay` 时 = 有效内高；`Flow::Down` 时 = 未用内高；然后应用 `min/max` 约束，减去 margin 高度
- `Size::Fixed`：直接返回
- `Size::Fit`：返回 NaN

### 3.6 `defer_walk_turtle()`（第 1932–1998 行）

延迟行走机制 —— 当行走宽度（`Flow::Right`）或高度（`Flow::Down`）为 `Size::Fill` 时，可以先占位但不分配具体尺寸：

1. 记录 `DeferredFill { weight, min, max }` 到 `deferred_fills` 列表
2. 推入 `DeferredWalk::Unresolved`，包含索引、位置、margin 和另一轴的尺寸
3. 在 `resolve_fill()` 中，当所有行走完成后，按权重比例分配剩余空间
4. `DeferredWalk::resolve()`（第 1221–1253 行）在编码阶段将 `Unresolved` 转换为 `Resolved(Walk)`，设置 `abs_pos` 为解析后的位置

此机制实现了类似 CSS Flexbox `flex-grow` 的效果。

### 3.7 `compute_final_size()`（第 1689–1743 行）

当海龟结束时，如果宽度/高度为 `Size::Fit`，此方法计算最终尺寸：

1. **宽度**：取 `used_width + padding.right`，然后应用 `Fit { min, max }` 约束
2. **高度**：取 `used_height + padding.bottom`，然后应用 `Fit { min, max }` 约束
3. 更新 `align_list` 中对应的 `BeginClip` 范围

### 3.8 `compute_max_height_from_ancestors()` / `compute_max_width_from_ancestors()`（第 1754–1815 行）

向上遍历海龟栈，查找最紧的 `Fit { max }` 约束。这对 `TextInput` 等控件特别有用——它们需要知道何时开始滚动，即使自身的 walk height 是无界的 `Fit`。

### 3.9 `wrap_turtle()`（第 2139–2151 行）

自动折行实现：
1. 记录旧位置
2. 调用 `turtle_new_line_internal()` 移动到新行
3. 计算位移向量 `shift = new_pos - old_pos`
4. 调用 `move_align_list()` 将已渲染的内容整体位移

### 3.10 `finish_row()` / `finish_row_bottom()` / `finish_row_center()`（第 2171–2321 行）

**`finish_row()`**：根据 `RowAlign` 类型分派：
- `RowAlign::Top`：不操作
- `RowAlign::Bottom`：调用 `finish_row_bottom()` 按基准线对齐
- `RowAlign::Center`：调用 `finish_row_center()` 垂直居中

**`finish_row_bottom()`**（第 2199–2263 行）：  
实现类似 CSS `vertical-align: baseline` 的文字排版对齐：
1. 计算当前行的 `descender` 和 `ascender`
2. 计算期望的行间距（基于 `prev_row_metrics` 的 descender + line_gap + 当前行的 ascender，乘以 `line_scale`）
3. 计算实际的行间距
4. 对行内每个行走，计算 `descender_shift + baseline_shift + line_spacing_shift`

**`finish_row_center()`**（第 2299–2321 行）：  
垂直居中行内元素。最⾼元素偏移为 0，较矮元素下移 `(row_height - walk_height) / 2`。

---

## 四、对齐系统（AlignEntry / align_list）

### 4.1 `AlignEntry` 枚举（第 33–46 行）

```rust
pub enum AlignEntry {
    Unset,                    // 未设置（用于无裁剪的根海龟）
    Area(Area),               // 已绘制的区域（实例或矩形区域）
    ShiftTurtle { area, shift, skip },  // 位移后的海龟
    SkipTurtle { skip },      // 跳过的海龟（裁剪已应用）
    BeginClip(Vec2d, Vec2d),  // 裁剪开始
    EndClip,                  // 裁剪结束
}
```

### 4.2 `move_align_list()`（第 2333–2398 行）

对齐列表的后处理引擎。遍历 `[start, end)` 范围内的对齐条目，应用 `(dx, dy)` 位移：

- **`Area::Instance`**：直接修改 GPU instance buffer 中的 `rect_pos` 和 `draw_clip` 字段，实现零成本渲染位移
- **`Area::Rect`**：修改 `RectArea` 的 `rect.pos` 和 `draw_clip`
- **`BeginClip`**：位移裁剪区域
- **`SkipTurtle` / `ShiftTurtle`**：跳过已处理的海龟

### 4.3 `clip_and_shift_align_list()`（第 2400–2476 行）

裁剪系统。遍历对齐列表，维护 `turtle_clips` 栈：
1. `BeginClip` 时将当前裁剪与父裁剪相交（取交集）
2. `EndClip` 时弹出裁剪栈
3. 对每个 `Area`，将裁剪写入 GPU 的 `draw_clip` 字段
4. `ShiftTurtle` 时计算位移并递归处理

---

## 五、辅助功能系统

### 5.1 `peek_walk_turtle()`（第 2114–2137 行）

无副作用的行走预览——计算行走的矩形而不分配空间。用于可见性判断（`walk_turtle_would_be_visible`）。

### 5.2 `walk_turtle_with_area()` / `end_turtle_with_area()`（第 2043–2057 行）

行走结束的同时注册 `Area`，供后续事件命中测试使用。

### 5.3 `emit_turtle_walk()` / `emit_turtle_walk_with_metrics()`（第 2092–2112 行）

供非行走类型的绘制操作（如背景色直接绘制）注册自己的已完成行走记录，使其也能参与对齐和布局。

### 5.4 `push_clip_rect()` / `pop_clip_rect()`（第 2497–2505 行）

手动裁剪栈操作，支持在行走之外任意推入/弹出裁剪区域。

### 5.5 `push_clip_rect_tracked()` / `update_clip_rect_at()`（第 1819–1832 行）

可跟踪的裁剪——返回 `align_list` 索引，后续可通过索引更新裁剪区域。

### 5.6 `shift_align_entries()`（第 2329–2331 行）

对外公开的对齐条目位移接口，供自定义布局场景使用。记录 start 和 end 值，调用 `move_align_list`。

### 5.7 `get_turtle_align_range()` / `shift_align_range()`（第 2478–2487 行）

获取和位移整个海龟的对齐范围。

---

## 六、DeferredFill 与权重分配系统（第 25–30 行，第 1144–1200 行）

```rust
struct DeferredFill {
    weight: f64,     // 权重
    max: Option<f64>,// 最大尺寸约束
    min: Option<f64>,// 最小尺寸约束
}
```

**`resolve_fill(index)`**（第 1175–1192 行）：

```
unresolved_length = inner_unused_length - total_resolved_length_to(index)
length = unresolved_length * weight / total_deferred_weight
length = clamp(length, min, max)
```

**惰性解析**：`resolve_fill` 按需解析，只在前 `count` 个已被解析时才解析新的。未解析的 fill 使用 `DeferredWalk::Unresolved` 存储，在最终编码时调用 `resolve()`。

---

## 七、编码阶段流程总结

1. **行走阶段**（`walk_turtle` / `begin_turtle`）：分配矩形，记录 `AlignEntry::Area`
2. **行结束阶段**（`finish_row`）：应用 `RowAlign` 位移
3. **海龟结束阶段**（`end_turtle`）：计算最终尺寸，应用 `Align` 偏移，应用裁剪
4. **后处理阶段**（`clip_and_shift_align_list`）：将裁剪写入 GPU 缓冲区

---

## 八、与 CSS Flexbox 的对比

| 概念 | CSS Flexbox | Makepad Turtle |
|------|-------------|----------------|
| 容器 | `display: flex` | `Layout` + `begin_turtle()` |
| 项目 | flex item | `walk_turtle()` |
| 方向 | `flex-direction` | `Flow::Right/Down/Overlay` |
| 换行 | `flex-wrap` | `Flow::Right { wrap: true }` |
| 对齐 | `justify-content` / `align-items` | `Align { x, y }` / `RowAlign` |
| 弹性 | `flex-grow` | `Size::Fill { weight }` |
| 间距 | `gap` | `Layout.spacing` / `Layout.wrap_spacing` |
| 内边距 | `padding` | `Layout.padding` |
| 外边距 | `margin` | `Walk.margin` |
| 基线对齐 | `vertical-align: baseline` | `RowAlign::Bottom` |
| 绝对定位 | `position: absolute` | `Walk.abs_pos` |
| 最小/最大尺寸 | `min-width` / `max-width` | `FitBound` / `Fill { min, max }` |
| scroll | `overflow: scroll` | `Layout.scroll` |

Makepad 的独特之处在于"光标推进"模型而非 Flexbox 的"主轴/交叉轴"计算，以及 `AlignEntry` 的 GPU 后处理位移（而非 CPU 端预计算）。
