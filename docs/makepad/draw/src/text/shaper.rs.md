# `draw/src/text/shaper.rs` — HarfBuzz 文本 Shaping

## 概述

这是 Makepad 的文本 shaping 模块（441 行），封装了 `rustybuzz`（HarfBuzz 的 Rust 绑定）的复杂 shaping 逻辑。核心职责：
1. **Unicode 双向文本（BiDi）解析**：将混合 LTR/RTL 文本分解为视觉顺序的运行
2. **字体回退**：主字体缺少字形时，递归尝试回退字体
3. **OpenType Shaping**：通过 `rustybuzz` 执行完整的字形替换和定位
4. **缩放/间距控制**：字距（letter-spacing）和词距（word-spacing）的后处理

## 核心类型

### `Shaper`（第 84-408 行）
Shaping 引擎。持有：
- `reusable_glyphs` — 回收 Vec 以减少分配
- `reusable_unicode_buffer` — 可复用的 Unicode 缓冲区
- `cached_features_source / cached_rb_features` — OpenType feature 转换缓存
- `cache_size / cached_params / cached_results` — LRU shaping 结果缓存

#### `Shaper::new(settings: Settings) -> Self`
构造方法。初始化可复用缓冲区、feature 缓存、以及具有 `settings.cache_size` 容量的 LRU 缓存。

#### `Shaper::get_or_shape(params: ShapeParams) -> Rc<ShapedText>`
顶级 shaping 入口。与 `Layouter::get_or_layout` 同样的缓存模式：
1. 若 `cache_size == 0`，直接 shaping 不缓存
2. 查缓存，命中返回
3. 未命中则淘汰最旧条目，执行 `self.shape(params)`，入缓存

#### `Shaper::shape(params: ShapeParams) -> ShapedText`
**核心 shaping 方法**。完整流程：

1. **快速路径（`is_definitely_ltr`）**：若文本不含任何可能的 RTL 字符，跳过 BiDi 解析，直接 `shape_recursive(text, ..., Direction::Ltr, 0, len)`

2. **BiDi 路径**：
   a. 用 `unicode_bidi::ParagraphBidiInfo::new(text, default_level)` 解析双向文本
   b. 若 `is_pure_ltr`（BiDi 确认全 LTR），仍使用单次 LTR shaping
   c. 否则调用 `visual_runs(0..text.len())` 获取视觉顺序的 level runs
   d. 对每个 run，按对应 direction（LTR 或 RTL）调用 `shape_recursive()`

3. **字距/词距后处理**：遍历所有 glyph，对每个 glyph 的 `advance_in_ems` 加上 `letter_spacing`，对空格字符再加上 `word_spacing`

4. 构建 `ShapedText { text, width_in_ems, glyphs }` 返回

#### `Shaper::shape_recursive(text, primary_font, fonts, features, direction, start, end, out_glyphs)`
**递归 shaping + 字体回退**（核心算法）。流程：

1. 取 `fonts[0]` 作为当前字体，其余为 `remaining_fonts`
2. 从 `reusable_glyphs` 获取回收的 glyph 容器，调用 `shape_step()` 执行 HarfBuzz shaping
3. 将 shaping 结果按 `cluster` 值分组（`group_by(同 cluster)`）
4. 遍历 glyph 分组：
   a. **若分组中存在 `.notdef`（id == 0）且有剩余回退字体**：收集连续含 `.notdef` 的分组范围 ([run_start, run_end])；根据 direction 计算缺失字形的逻辑字节范围；递归调用 `shape_recursive()` 用下一个回退字体重新 shaping
   b. **若分组中有 `.notdef` 但已无回退字体**：使用主字体的 `.notdef` 字形（显示为方块或问号占位符）
   c. **正常字形**：直接输出
5. 清空并回收 glyph 容器

#### `Shaper::shape_step(text, font, features, direction, start, end, out_glyphs)`
**单次 HarfBuzz shaping 调用**。流程：
1. 从 `reusable_unicode_buffer` 获取回收的 UnicodeBuffer，设置方向
2. 对 `text[start..end]` 按字素（grapheme）迭代，每个字素拆分为 Unicode codepoints 加入 buffer。`cluster` 设为在原始文本中的字节偏移
3. feature 转换：将调用者的 `(tag_u32, value)` 对转换为 `rustybuzz::Feature`，缓存到 `cached_features_source/cached_rb_features`。对比使用内容比较而非指针比较，避免因 `Rc` 释放重用地址导致的误命中
4. 调用 `font.with_rustybuzz_face(|face| rustybuzz::shape(face, ...))` 执行 shaping
5. 从输出的 glyph info + positions 构建 `ShapedGlyph` 列表

---

### `ShapeParams`（第 415-424 行）
Shaping 参数。包含文本（Substr）、字体列表（`Rc<[Rc<Font>]>`）、方向、字距/词距（`Ems`）、OpenType feature 列表。

### `ShapedText` / `ShapedGlyph`（第 426-441 行）
Shaping 输出。`ShapedText` 包含文本引用、总宽度（em 单位）和 glyph 列表。`ShapedGlyph` 包含字体引用、字形 ID、cluster（原始文本偏移）、前进宽度、水平/垂直偏移。

---

### 辅助类型

#### `Ems`（第 53-74 行）
`f32` 的包装类型，实现了 `Hash + Eq`（通过 `to_bits()` 位比较），用于 shapng 参数中的间距值。

#### `is_definitely_ltr(text: &str) -> bool`（第 26-50 行）
**快速 LTR 检测**。纯 ASCII 文本（`is_ascii()` 使用 SIMD 加速扫描）必然为 LTR。非 ASCII 文本检查 Unicode 范围：
- BMP 中的希伯来语/阿拉伯语区块 (0x0590-0x08FF)
- 字母呈现形式 (0xFB1D-0xFDFF)
- 阿拉伯呈现形式 B (0xFE70-0xFEFF)
- SMP 中的古文字 (0x10800-0x10FFF)
- 现代 SMP RTL 区块 (0x1E800-0x1EFFF)

若文本不含这些区块中的任何字符，返回 `true`（跳过 BiDi）。**假阳性（含非 RTL 字符的 RTL 区块被跳过）不会导致渲染错误**——BiDi 会纠正。假阴性仅增加一次不必要的 BiDi 解析开销。
