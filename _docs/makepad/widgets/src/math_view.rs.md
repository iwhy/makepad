# `math_view.rs` — LaTeX 数学公式渲染 Widget

## 概述

`MathView` 是 Makepad 框架中用于渲染 LaTeX 数学公式的 widget。它依赖 `makepad_latex_math` 库完成 LaTeX 解析和数学排版布局，然后将布局结果中的字形（Glyph）、横线（Rule）和矩形（Rect）转换为 `DrawGlyph` 的矢量图形来绘制，最终实现高保真的数学公式显示。公式渲染使用 NewCMMath 字体（Latin Modern Math 的变体），适合科学计算、笔记、文档预览等场景。

---

## 类型定义

### `struct MathComponent`（私有辅助结构体）

```rust
#[derive(Clone, Copy, Debug)]
struct MathComponent {
    shape_id: GlyphShapeId,
    origin: Vec2f,
    size: Vec2f,
}
```

**功能**：存储一个已构建好的字形/图形在 `DrawGlyph` 内部的句柄及其位置尺寸信息。

- `shape_id: GlyphShapeId`：`DrawGlyph` 中已提交（commit）的形状的唯一标识，用于后续绘制引用。
- `origin: Vec2f`：该形状在布局坐标系中的原点坐标（x, y），单位为像素。
- `size: Vec2f`：该形状的包围盒尺寸（宽, 高）。

**实现逻辑**：该结构体在 `push_component` 辅助函数中从 `DrawGlyph::shape(shape_id)` 查询得到 `origin` 和 `size` 后填充。它是一个纯内部优化结构，将渲染时需要的所有元数据提前缓存，避免每帧重复查询。

---

### `struct MathView`

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct MathView {
    #[uid] uid: WidgetUid,
    #[source] source: ScriptObjectRef,
    #[walk] walk: Walk,
    #[redraw] #[live] draw_glyph: DrawGlyph,
    #[live] draw_text: DrawText,
    #[live] text: String,
    #[live] color: Vec4,
    #[live(11.0)] font_size: f64,
    #[live(-2.0)] baseline_offset: f64,
    #[rust] old_text: String,
    #[rust] old_font_size: f64,
    #[rust] old_font_family_id: Option<FontFamilyId>,
    #[rust] layout_cache: Option<latex_math::LayoutOutput>,
    #[rust] components: Vec<MathComponent>,
    #[rust] debug_reason: Option<String>,
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `uid` | `WidgetUid` | Widget 唯一标识，由 `#[uid]` 宏自动管理 |
| `source` | `ScriptObjectRef` | 脚本对象引用，支持 `Script` derive 的运行时反射 |
| `walk` | `Walk` | 布局行走参数，最终会被布局尺寸覆盖 |
| `draw_glyph` | `DrawGlyph` | **核心渲染器**：用于存储和绘制矢量字形形状 |
| `draw_text` | `DrawText` | 备用文本渲染器：公式解析失败时显示调试信息 |
| `text` | `String` | 用户输入的 LaTeX 数学表达式源码 |
| `color` | `Vec4` | 公式颜色（RGBA），可在脚本中动态修改 |
| `font_size` | `f64` | 字体大小（像素），默认 11.0 |
| `baseline_offset` | `f64` | 基线垂直偏移调整，默认 -2.0 |
| `old_text` | `String` | 缓存的上一帧文本，用于变更检测 |
| `old_font_size` | `f64` | 缓存的上一帧字号，用于变更检测 |
| `old_font_family_id` | `Option<FontFamilyId>` | 缓存的上一帧字体族 ID |
| `layout_cache` | `Option<latex_math::LayoutOutput>` | LaTeX 布局结果的缓存 |
| `components` | `Vec<MathComponent>` | 已构建的图形组件列表，用于每帧快速绘制 |
| `debug_reason` | `Option<String>` | 调试/错误原因，非空时回退为文本渲染 |

---

## 脚本模块注册

```rust
script_mod! {
    mod.widgets.MathViewBase = #(MathView::register_widget(vm))
    mod.widgets.MathView = set_type_default() do mod.widgets.MathViewBase {
        // 默认属性：Fit 尺寸、白色、11px 字号、NewCMMath-Regular 字体
    }
}
```

**实现逻辑**：

1. **注册基类**：`MathViewBase` 通过 `MathView::register_widget(vm)` 将 Rust 结构体注册到脚本运行时，使脚本系统能够识别和实例化该 widget。

2. **默认值设置**：使用 `set_type_default()` 创建 `MathView` 样式，继承 `MathViewBase` 的所有字段，然后用 `+: ` 语法合并覆盖默认值：
   - `width: Fit, height: Fit` — 自适应内容尺寸。
   - `color: #fff` — 默认白色文本。
   - `font_size: 11.0` — 默认 11 像素字号。
   - `baseline_offset: -2.0` — 基线向上偏移 2 像素，使公式与周围文本对齐更美观。
   - `draw_glyph` 的 `aa_pad_px: 1.0` — 抗锯齿填充 1 像素。
   - `draw_text` 的 `text_style` 配置为 Latin Modern Math 字体（NewCMMath-Regular.otf），作为回退调试字体。

---

## Trait 实现

### `impl Widget for MathView`

#### `fn handle_event`

```rust
fn handle_event(&mut self, _cx: &mut Cx, _event: &Event, _scope: &mut Scope) {}
```

**实现逻辑**：当前版本为**无操作实现**。`MathView` 是完全自包含的静态组件，不接收鼠标、键盘或触摸事件。所有状态变更通过父 widget 调用 `set_text` 触发。

---

#### `fn draw_walk`

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, _scope: &mut Scope, mut walk: Walk) -> DrawStep
```

**绘制主流程（共 6 步）**：

1. **编译触发**：调用 `self.compile_math(cx)`。该方法是脏检测和延迟编译的入口，只有在文本、字号或字体族发生变化时才真正执行 LaTeX 解析和图形构建。

2. **布局空值处理**：检查 `self.layout_cache`。如果为 `None`（解析失败或文本为空），则根据 `self.debug_reason` 的内容构建调试文本 `[reason]`，使用 `draw_text.draw_walk` 回退为普通文本显示。若 `debug_reason` 也为空（空文本），则直接结束绘制。

3. **覆盖行走尺寸**：从布局缓存中读取 `layout.width` 和 `layout.height`，将 `walk.width` 和 `walk.height` 设为 `Size::Fixed`。这使得 `MathView` 的占位空间与其渲染内容精确匹配。同时将 `walk.margin.top` 加上 `baseline_offset`，实现基线微调。

4. **行走与区域计算**：调用 `cx.walk_turtle(walk)` 获得矩形区域 `bounds`，`cx` 的绘图状态前进到该区域。

5. **绘制每个组件**：遍历 `self.components` 数组，对每个 `MathComponent`：
   - 通过 `self.draw_glyph.shape(component.shape_id)` 获取 `DrawGlyphShape` 的引用（包含图层列表）。
   - 克隆该形状的 `layers`，将每个图层的颜色乘以 widget 的 `color`，实现颜色叠加（白色图层 × 用户颜色 = 最终颜色）。
   - 调用 `self.draw_glyph.draw_layers_abs` 以绝对坐标绘制每个图层，坐标原点为 `bounds.pos + component.origin`，尺寸为 `component.size`。

6. **结束绘制**：返回 `DrawStep::done()`。

---

#### `fn set_text`

```rust
fn set_text(&mut self, cx: &mut Cx, text: &str)
```

**实现逻辑**：将 `text` 字段赋值为用户传入的字符串，然后调用 `self.redraw(cx)` 触发重绘。实际的 LaTeX 解析和图形构建不会在此处立即执行，而是推迟到下一帧 `draw_walk` 中的 `compile_math` 方法中进行，利用脏检测机制判断是否真的需要重新编译。

---

## MathView 私有实现

### `fn compile_math`

```rust
fn compile_math(&mut self, cx: &mut Cx2d)
```

**核心编译方法（共 7 个阶段）**：

1. **字体加载与脏检测**：
   - 从 `draw_text.text_style` 获取 `font_family_id`。
   - 调用 `ensure_fonts_loaded` 确保该字体族已加载到上下文中。
   - 比较 `text`、`font_size` 和 `font_family_id` 是否与缓存值 `old_text`、`old_font_size`、`old_font_family_id` 一致，且 `layout_cache` 已存在（或文本为空）。
   - 如果所有条件都满足（无变化且缓存有效），直接 return，跳过编译。

2. **更新缓存标记**：将 `old_text`、`old_font_size`、`old_font_family_id` 更新为当前值。

3. **空文本处理**：如果 `self.text` 为空，清除 `layout_cache`、`components`、`draw_glyph` 中的所有形状和 `debug_reason`，然后 return。

4. **获取布局字体**：
   - 从 `cx.fonts.borrow_mut()` 获取字体系统的可变引用。
   - 通过 `get_or_load_font_family(font_family_id)` 获取字体族。
   - 取字体族中的第一个字体（即 NewCMMath-Regular）作为布局引擎的字体。
   - 如果无法获取（字体文件缺失），设置 `debug_reason = "missing-math-font"`，清除所有缓存，return。

5. **LaTeX 解析与布局**：
   - 调用 `latex_math::parse(&self.text)` 将 LaTeX 源码解析为抽象语法树节点 `nodes`。
   - 计算布局字号：`font_size * 1.75`。放大系数 1.75 是为了将像素单位的字号映射到数学字体的设计空间单位，使公式渲染大小与普通文本协调。
   - 调用 `latex_math::layout(&nodes, font_data, layout_size, MathStyle::Display)` 执行数学排版。
   - 如果排版失败，设置 `debug_reason = "math-layout-failed"`，清除所有缓存，return。

6. **构建图形形状**：
   - 调用 `self.draw_glyph.clear_shapes()` 清空上一帧的图形数据。
   - 遍历 `layout.items`，每个 `LayoutItem` 有三种变体：
     - **`LayoutItem::Glyph(glyph)`**：调用 `build_glyph_shape` 将字体的轮廓数据（MoveTo/LineTo/QuadTo/CurveTo/Close）转换为 `DrawGlyph` 的矢量路径，然后通过 `push_component` 保存组件元数据。
     - **`LayoutItem::Rule(rule)`**：调用 `build_rule_shape`（内部委托给 `build_rect_shape`）构建一个填充矩形，对应 LaTeX 中的分数线（`\frac` 的水平线）或根号（`\sqrt` 的横线）。
     - **`LayoutItem::Rect(rect)`**：调用 `build_rect_shape` 直接构建矩形，对应 LaTeX 中的方框或定界符的矩形部分。

7. **缓存更新**：将 `layout` 保存到 `self.layout_cache`，`components` 保存到 `self.components`，`debug_reason` 设为 `None`。

---

## 辅助函数

### `fn build_glyph_shape`

```rust
fn build_glyph_shape(
    dg: &mut DrawGlyph,
    font: &Font,
    glyph: &LayoutGlyph,
    layout_ascent: f32,
) -> Option<GlyphShapeId>
```

**实现逻辑（共 6 步）**：

1. **坐标计算**：`glyph_x = glyph.x`（水平偏移），`glyph_y = layout_ascent + glyph.y`（垂直偏移，从基线向上）。`font_scale = glyph.size / font.units_per_em()` 计算从字体设计空间到像素空间的缩放比例。

2. **字形轮廓查询**：调用 `font.glyph_outline(glyph.glyph_id)` 获取字形的矢量轮廓数据。如果该字形在字体中不存在（如某些字形缺失），返回 `None`。

3. **开始形状构建**：调用 `dg.begin_shape()` 开始一个新的矢量形状，然后调用 `dg.set_color(1,1,1,1)` 设置填充颜色为白色（实际颜色在绘制时通过 `color` 字段叠加）。

4. **轮廓路径转换**：遍历 `outline.commands()`，将字体轮廓中的每个命令转换为 `DrawGlyph` 的路径操作：
   - **`MoveTo(p)`**：`dg.move_to(x, y)`，移动到起始点。坐标转换为 `glyph_x + p.x * font_scale`（水平）和 `glyph_y - p.y * font_scale`（垂直，因为字体坐标系的 Y 轴向上而屏幕 Y 轴向下，所以取反）。
   - **`LineTo(p)`**：`dg.line_to(x, y)`，画直线段到指定点。
   - **`QuadTo(c, p)`**：`dg.quad_to(cx, cy, px, py)`，画二次贝塞尔曲线，`c` 为控制点，`p` 为终点。
   - **`CurveTo(c1, c2, p)`**：`dg.bezier_to(c1x, c1y, c2x, c2y, px, py)`，画三次贝塞尔曲线，`c1`、`c2` 为两个控制点，`p` 为终点。
   - **`Close`**：`dg.close()`，闭合当前子路径。

5. **填充与提交**：调用 `dg.fill_layer()` 将当前路径填充为一个图层，然后调用 `dg.commit_shape(None)` 提交形状到 `DrawGlyph` 的内部存储，返回 `Some(GlyphShapeId)`。

---

### `fn build_rule_shape`

```rust
fn build_rule_shape(
    dg: &mut DrawGlyph,
    rule: &LayoutRule,
    layout_ascent: f32,
) -> Option<GlyphShapeId>
```

**实现逻辑**：这是一个轻量级委托函数。从 `rule` 中提取 `x`、`y`、`width`、`height`，将 `y` 通过 `layout_ascent + rule.y` 转换到屏幕坐标，然后将全部参数转发给 `build_rect_shape`。`LayoutRule` 通常表示分数线（`\frac{a}{b}` 中分子和分母之间的水平线）。

---

### `fn build_rect_shape`

```rust
fn build_rect_shape(dg: &mut DrawGlyph, x: f32, y: f32, w: f32, h: f32) -> Option<GlyphShapeId>
```

**实现逻辑**：
1. 调用 `dg.begin_shape()` 开始形状构建。
2. 调用 `dg.set_color(1,1,1,1)` 设置白色填充。
3. 调用 `dg.rect(x, y, w, h)` 构造一个矩形路径。
4. 调用 `dg.fill_layer()` 填充该矩形。
5. 调用 `dg.commit_shape(None)` 提交并返回 `Some(GlyphShapeId)`。

该函数是 `build_rule_shape` 和 `LayoutItem::Rect` 的底层实现。

---

### `fn push_component`

```rust
fn push_component(dg: &DrawGlyph, shape_id: GlyphShapeId, components: &mut Vec<MathComponent>)
```

**实现逻辑**：
1. 调用 `dg.shape(shape_id)` 获取已提交形状的元数据（`origin`、`size`）。如果句柄无效（`None`），直接 return。
2. 创建一个 `MathComponent` 结构体，填入 `shape_id`、查询到的 `origin` 和 `size`。
3. 将该组件压入 `components` 向量，供后续 `draw_walk` 阶段的快速绘制使用。
4. 这种分离设计（构建时一次性计算所有元数据，绘制时直接引用）避免了在绘制循环中反复查询 `DrawGlyph` 内部数据结构的开销。

---

## 整体数据流

```
用户设置 text
    ↓
set_text() → redraw() 触发下一帧
    ↓
draw_walk() → compile_math()
    ├─ dirty? → 跳过（使用缓存）
    └─ dirty! → latex_math::parse() → latex_math::layout()
                ↓
        遍历 LayoutItems
        ├─ Glyph → build_glyph_shape（字体轮廓 → DrawGlyph 路径）
        ├─ Rule  → build_rule_shape → build_rect_shape
        └─ Rect  → build_rect_shape
                ↓
        push_component 填充 components
                ↓
        layout_cache = Some(layout)
    ↓
draw_walk() 继续
    ├─ 使用 layout_cache 设置 walk 尺寸
    └─ 遍历 components → draw_layers_abs 逐个绘制
```

## 用途与场景

- **科学笔记应用**：结合 `TextInput` 和 `Markdown` 渲染器，可构建支持 LaTeX 内联公式的笔记编辑器。
- **公式预览**：在 LaTeX 编辑器的侧边或弹窗中实时预览数学公式效果。
- **数学课件/幻灯片**：作为 `SlidesView` 的子 widget 展示数学内容。
- **调试回退**：当字体缺失或公式语法错误时，通过 `debug_reason` 以纯文本 `[reason]` 形式给出错误提示，避免完全空白。
