# `draw/src/text/layouter.rs` — 文本布局引擎（核心文件）

## 概述

这是 Makepad 文本渲染的核心文件（1339 行），实现了完整的文本布局管线：从 `Layouter` 公共 API 到内部的 `LayoutContext` 逐行布局、`Fitter` 分段适配、以及 `LaidoutText`/`LaidoutRow`/`LaidoutGlyph` 的输出数据结构。

文本渲染管线的完整流程：

```
原始文本 + Style + LayoutOptions
  │
  ▼
Layouter::get_or_layout()  [缓存查找]
  │
  ▼
LayoutContext::layout_multiline()
  │  split('\n') 按显式换行分段
  │  → 每段调用 layout() → layout_by_word() / layout_directly()
  │  → apply_ellipsis_truncation()（可选）
  ▼
LaidoutText { rows: [LaidoutRow], glyphs: ... }
```

---

## 常量

### `LPXS_PER_INCH` / `PTS_PER_INCH`
定义逻辑像素（LPXS）与物理点（PT）的转换关系：96 DPI 逻辑分辨率、72 点/英寸。`font_size_in_pts` → `font_size_in_lpxs` 的转换即 `font_size_in_pts * 96.0 / 72.0`。

### 缓存阈值常量
- `LAYOUT_CACHE_MAX_TEXT_LEN = 512`：超过此长度的文本不缓存
- `LAYOUT_CACHE_MULTILINE_TEXT_LEN = 192` + `LAYOUT_CACHE_MULTILINE_LINE_COUNT = 4`：中长度多行文本也不缓存（调试场景的 profiler 输出等高频变化文本）

---

## 核心类型

### `Layouter`（第 36-135 行）
文本布局的公共入口点。持有：
- `loader: Loader` — 字体加载子系统（间接持有 Shaper + Rasterizer）
- `cache_size` / `cached_params` / `cached_results` — LRU 布局缓存（`VecDeque < FxHashMap`）

#### `Layouter::should_cache_text(text: &str) -> bool`
判断文本是否适合缓存。逻辑：
1. 若长度 > 512 字节，不缓存（大文本命中率低且缓存开销大）
2. 若长度 > 192 字节且行数 >= 4，不缓存（这是 profiler/调试输出的典型特征，每帧都在变）
3. 否则可以缓存

#### `Layouter::new(settings: Settings) -> Self`
构建 Layouter。初始化 Loader（内部构建 Shaper + Rasterizer），按 `settings.cache_size` 分配缓存容量。

#### `Layouter::get_or_layout(params: impl LayoutParams) -> Rc<LaidoutText>`
布局的核心入口。策略：
1. 若 `cache_size == 0` 或 `should_cache_text()` 返回 false，直接布局不缓存
2. 否则查 `cached_results`，命中则返回 `Rc` 克隆
3. 未命中且缓存已满：淘汰最旧的参数对
4. 执行 `self.layout(params)` 生成结果，插入缓存后返回

#### `Layouter::layout(params: OwnedLayoutParams) -> LaidoutText`
内部布局方法。从 Loader 加载目标 `FontFamily`，创建 `LayoutContext` 并调用 `layout_multiline()`。

#### `define_font_family` / `set_font_family_definition` / `define_font`
委托给 `self.loader`。`set_font_family_definition` 在更新定义时会清空整个布局缓存（因为行宽可能因字体变化而改变）。

---

### `Settings`（第 137-181 行）
布局器的全部配置参数。`Default::default()` 会递归构建默认的 `loader::Settings`、`shaper::Settings`、`rasterizer::Settings`（包含 `Sdfer`/`Msdfer` 的 padding/radius/cutoff 参数，以及 MSDF 分辨率策略、复杂性阈值等）。

`atlas_size` 默认为 2048×2048，可通过环境变量 `MAKEPAD_TEXT_ATLAS_SIZE` 覆盖（支持 `"1024"` 正方形或 `"1024x2048"` 矩形，范围 256~8192）。

---

### `LayoutContext`（第 214-561 行）
**单次布局操作的状态机**，持有：
- `font_family: Rc<FontFamily>` — 目标字体系列（含字体回退链）
- `text: Substr` — 布局中的全部文本（子串引用）
- `style: Style` — 排版样式（字体大小、颜色）
- `options: LayoutOptions` — 布局选项（最大宽度、换行、对齐等）
- `current_point_in_lpxs: Point<f32>` — 当前写入光标位置（逻辑像素空间）
- `current_row_start / current_row_end` — 当前行在原始文本中的字节区间
- `rows: Vec<LaidoutRow>` / `glyphs: Vec<LaidoutGlyph>` — 累积的输出

#### `LayoutContext::new(...) -> Self`
初始化上下文。`current_point_in_lpxs.x` 设为 `first_row_indent_in_lpxs`（首行缩进）。

#### `LayoutContext::layout_multiline(mut self) -> LaidoutText`
**整个布局的顶层循环**。流程：
1. 对 `self.text.split('\n')` 枚举每一"逻辑行"
2. 非首行先调用 `finish_current_row(true)` 结束上一行（带换行标志）
3. 若已超过 `max_rows`，跳出循环
4. 对每个逻辑行调用 `self.layout(len)` 进行断行布局
5. 布局完成后，若有 ellipsis 配置则调用 `apply_ellipsis_truncation()`，否则直接 `finish_with(false)`

#### `LayoutContext::layout(len: usize)`
单行布局的调度中枢。根据 `wrap` 选项决定策略：
- 无限宽（`wrap = false` 且 `max_width` 未设置）：调用 `layout_directly(len)` 一次排完整个逻辑行
- 限制宽度：调用 `layout_by_word(len)` 逐词适配

#### `LayoutContext::layout_by_word(len: usize)`
**按词断行**。核心逻辑：
1. 创建 `Fitter`（按 `SegmentKind::Word` 分段）
2. 循环 `fitter.fit(remaining_width)`：
   - 返回 `Some(text)` → 调用 `append_text()` 将字形追加到当前行
   - 返回 `None`（当前词放不下）：
     - 若下一个 segment 全是空白 → 调用 `layout_directly()` 强制放下（空白不可见，无需断行）
     - 若当前行为空且不是首行缩进续行 → 调用 `layout_by_grapheme()` 按字素级断行（长单词/URL 不得不打断）
     - 否则 → `finish_current_row(false)` 换行，下轮循环继续

#### `LayoutContext::layout_by_grapheme(len: usize)`
**按字素级断行**（迫不得已时的最后手段）。与 `layout_by_word` 类似，但 `Fitter` 按 Unicode 字素（用户感知字符）分段。这确保即使打断长单词，也不会在字素中间断开。

#### `LayoutContext::layout_directly(len: usize)`
**直接布局**（不换行）。调用 `font_family.get_or_shape()` 对整段文本做一次 shaping，然后 `append_text()`。

#### `LayoutContext::append_text(text: &ShapedText)`
将 ShapedText 的字形追加到当前行。对每个 ShapedGlyph：
1. 创建 `LaidoutGlyph`，设 `cluster` 为相对于当前行起始的偏移
2. `origin_in_lpxs.x` 设为当前光标位置
3. 光标前进 `glyph.advance_in_lpxs()`
4. 累加 `current_row_end`

#### `LayoutContext::finish_current_row(newline: bool)`
**终结当前行**：
1. 取字体的 ascender/descender/line_gap 数据（用第一个字体，乘以 `font_size_in_lpxs`）
2. 从 `self.glyphs` 中 `mem::take` 出属于该行的字形
3. 构建 `LaidoutRow`，计算 `origin_in_lpxs.y`：累加上一行的 `line_spacing_in_lpxs()`
4. `origin_in_lpxs.x = align * (max_width - row.width)` 实现水平对齐（0=左对齐，0.5=居中，1=右对齐）
5. 重置 `current_point_in_lpxs`，更新 `current_row_start/end`

#### `LayoutContext::apply_ellipsis_truncation(mut self) -> LaidoutText`
省略号截断的后处理。触发条件：
1. `max_rows` 限制且文本被截断（`current_row_end` 未覆盖全文）
2. `ellipsis = true`、`wrap = false`、单行宽度超过 `max_width_in_lpxs`

实现步骤：
1. 调用 `finish_current_row_if_pending()` 确保已处理的字形都入行
2. 若触发条件 1，`truncate(max_rows)` 砍掉超出行
3. 若 `ellipsis = true`，调用 `truncate_last_row_with_ellipsis(max_width)`

#### `LayoutContext::truncate_last_row_with_ellipsis(max_width: f32)`
**用省略号替换最后一行尾部内容**：
1. 对 `"…"` 做 shaping 得到省略号宽度
2. 从末尾逐 glyph 弹出，直到 `剩余宽度 + 省略号宽度 <= max_width`
3. 再弹出尾部空白字符的字形（避免省略号前有空格）
4. 追加省略号字形（`cluster` 设为文本末尾之后，超出原始范围的哨兵值）

---

### `Fitter`（第 563-662 行）
**分段-适配-消费** 工具，支持按词和按字素两种分段模式。

#### `Fitter::new(text, font_family, font_size, segment_kind) -> Self`
构造时完成所有预计算：
1. 按 `segment_kind` 将文本分段：`SegmentKind::Word` 用 `split_word_bounds()`；`SegmentKind::Grapheme` 用 `graphemes(true)`
2. 若为 `Word` 模式，调用 `merge_segments_for_line_breaking()` 合并不可断字位置
3. 对每个 segment，调用 `font_family.get_or_shape()` 做 shaping，计算宽度（`width_in_ems * font_size`），存入 `widths_in_lpxs`

#### `Fitter::fit(wrap_width_in_lpxs: f32) -> Option<Rc<ShapedText>>`
**二分搜索最佳适配段数**。优化关键：
- 使用预计算的 `widths_in_lpxs` 前缀和估算宽度，避免在搜索过程中反复调用 `get_or_shape()`（累积子串会 miss 缓存，导致哈夫布兹重排）
- 搜索范围 `[1, lens.len()]`，用 `can_fit()` 判定
- 找到最大可适配段数后，调用 `font_family.get_or_shape()` 获取精确的已 shaping 文本
- 消费（drain）对应 segment 的 lens 和 widths

#### `Fitter::can_fit(count, wrap_width) -> bool`
用预计算宽度的前缀和估算 `count` 段的总宽度是否 <= `wrap_width`。不精确但够用（忽略跨词字距调整）。

#### `Fitter::pop() -> usize`
弹出第一段并返回其长度。用于 `layout_directly()` 消费不可断开的段。

---

### `merge_segments_for_line_breaking`（第 678-714 行）
线断合法化合并，遵循 UAX#14 / CSS Text Module Level 3 规范。
- **Pass 1**：将"断前禁止"字符（尾部/闭合标点 `. , ; ! ? ) ] } … % °`）合并到前一段，避免它们单独成行
- **Pass 2**：将"断后禁止"字符（开头标点 `( [ { ' " ‹ «`）合并到后一段，避免它们留在行尾

---

### `LayoutParams` trait（第 752-816 行）
布局参数的抽象接口，支持 `OwnedLayoutParams`（拥有型）和 `BorrowedLayoutParams`（借用型）。实现 `Eq` + `Hash` 用于缓存键：
- `Hash` 通过组合 `text() + style() + options()` 实现
- `Eq` 按字段逐位比较（浮点数用 `to_bits()`）

---

### `Style`（第 853-892 行）
排版样式：`font_family_id`（字体系列标识）、`font_size_in_pts`（点数字号）、`color`（可选颜色）。
- `font_size_in_lpxs()`：将点转换为逻辑像素 `pts * 96.0 / 72.0`

---

### `LayoutOptions`（第 894-960 行）
布局选项：
- `first_row_indent_in_lpxs`：首行缩进
- `max_width_in_lpxs`：最大行宽（`None` = 不限制）
- `wrap`：是否换行
- `align`：水平对齐（0.0 左对齐，0.5 居中，1.0 右对齐）
- `line_spacing_scale`：行距缩放因子
- `max_rows`：最大行数限制
- `ellipsis`：截断时是否追加省略号

---

### `LaidoutText`（第 962-1095 行）
布局完成的输出结构。包含原始文本引用、整体尺寸（`size_in_lpxs`）、行列表、以及截断标志。

**光标 ↔ 位置转换方法**：

#### `cursor_to_position(cursor: Cursor) -> CursorPosition`
将逻辑光标（字符串索引）转换为行坐标。过程：
1. `cursor_to_row_index()` 遍历行，找到光标索引所在的行（优先当前行，`prefer_next_row` 控制上/下行偏好）
2. 调用 `row.index_to_x_in_lpxs()` 计算行内 x 坐标

#### `point_in_lpxs_to_cursor(point) -> Cursor`
将像素坐标转换为光标。过程：
1. `y_in_lpxs_to_row_index()` 按 y 坐标定位行（取最近行间中点）
2. `position_to_cursor()` 调用 `row.x_in_lpxs_to_index()` 获取字符索引

#### `selection_rects(selection: Selection) -> Vec<SelectionRect>`
计算选择区域的高亮矩形列表。根据起止行所在位置：
- **同行**：一个矩形
- **跨行**：首行从 start_x 到行尾 + 中间行整行 + 末行从行首到 end_x

---

### `LaidoutRow`（第 1103-1187 行）
已布局的行。包含：
- `origin_in_lpxs`：行原点（x 含对齐偏移，y 为累积行高）
- `text`：该行的文本子串
- `newline`：是否由显式换行符结束
- `width_in_lpxs`：行实际宽度
- `ascender/descender/line_gap_in_lpxs`：排版度量
- `line_spacing_scale`：行距缩放
- `glyphs`：该行的字形列表

#### `line_spacing_in_lpxs(next_row) -> f32`
计算当前行与下一行之间的行距：`(line_gap - descender + next_row.ascender) * scale`

#### `x_in_lpxs_to_index(x) -> usize`
像素坐标 → 字符索引（用于点击定位）。算法：
1. 按 `cluster` 值对 glyphs 分组（一个集群可能对应多个 glyph，如合字）
2. 对每个集群组，计算其总宽度和字素数
3. 将集群宽度均匀分给每个字素（`grapheme_width = cluster_width / grapheme_count`）
4. 遍历字素，找到点击 x 坐标所在的字素边界（取中点判断）

#### `index_to_x_in_lpxs(index) -> f32`
字符索引 → 像素坐标（用于光标绘制）。算法同上对称——遍历字素，返回命中索引对应的 x 坐标。

---

### `LaidoutGlyph`（第 1189-1225 行）
布局后的单个字形。包含：
- `origin_in_lpxs`：绘制原点
- `font`：所属字体（Rc 引用，支持跨字体回退的混合）
- `font_size_in_lpxs`：渲染字号
- `color`：可选颜色覆盖
- `id: GlyphId`：字形 ID
- `cluster`：在文本中的字节偏移
- `advance_in_ems`：前进宽度（相对 em 单位）
- `offset_in_ems`：水平偏移

#### `advance_in_lpxs() / offset_in_lpxs()`
将 em 单位转换为逻辑像素。

#### `ascender_in_lpxs() / descender_in_lpxs() / line_gap_in_lpxs()`
从字体获取度量值，乘以字号。

#### `rasterize(dpxs_per_em) - > Option<RasterizedGlyph>`
委托给 `font.rasterize_glyph()`，最终调用 Rasterizer。

---

## 测试

第 1227-1339 行的测试模块覆盖：
1. `parse_text_atlas_size_value`：环境变量解析的各种边界
2. `merge_segments_for_line_breaking`：各种标点合并场景（尾部标点、开头标点、组合、中英文混合）
3. `should_cache_text`：缓存决策的正确性（短文本、长文本、多行调试文本）
