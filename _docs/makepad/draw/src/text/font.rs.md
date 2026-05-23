# `draw/src/text/font.rs` — 核心字体抽象

## 概述

这是 Makepad 字体系统的核心类型 `Font`（261 行），封装了单个字体的完整能力：字形度量、轮廓提取、光栅化、以及嵌入式位图字形访问。

## 核心类型

### `FontId`（第 22-35 行）
字体唯一标识符（`u64` newtype）。可以从 `u64` 直接构造，也可以从 `&str` 通过字符串驻留指针（`Intern`）构造。`Eq + Hash + Copy` 用于缓存键。

### `GlyphId = u16`（第 207 行）
字形 ID 的类型别名。

### `Font`（第 37-206 行）
单字体的核心表示。持有：
- `id: FontId` — 标识
- `rasterizer: Rc<RefCell<Rasterizer>>` — 共享光栅化器引用
- `face: FontFace` — 字体表面（封装 ttf_parser/rustybuzz Face）
- `units_per_em: f32` — EM 方格大小
- `ascender_in_ems / descender_in_ems / line_gap_in_ems` — 排版度量
- `cached_glyph_outlines: RefCell<FxHashMap<GlyphId, Option<GlyphOutline>>>` — 字形轮廓缓存

#### `Font::new(id, rasterizer, face, ascender_fudge, descender_fudge) -> Self`
构造 Font。从 `ttf_parser::Face` 提取度量数据（units_per_em、ascender、descender、line_gap），全部除以 `units_per_em` 转换为 em 相对值。`ascender_fudge` 和 `descender_fudge` 用于微调字体行高（如通过字体定义传递的校正值）。

#### `with_ttf_parser_face<R>(f) -> R`
通过闭包访问 `ttf_parser::Face`。委托给 `FontFace::with_ttf_parser_face()`。

#### `with_rustybuzz_face<R>(f) -> R`
通过闭包访问 `rustybuzz::Face`。委托给 `FontFace::with_rustybuzz_face()`。

#### `glyph_outline(glyph_id) -> Option<GlyphOutline>`
**获取字形矢量轮廓**。流程：
1. 查 `cached_glyph_outlines` 缓存，命中则返回克隆
2. 调用 `with_ttf_parser_face` 获取字形轮廓：
   a. 创建 `glyph_outline::Builder`
   b. 调用 `face.outline_glyph()`, `Builder` 回调收集 MoveTo/LineTo/QuadTo/CurveTo/Close 指令
   c. 获取 ttf_parser 返回的边界框，构建 `Rect<f32>`
   d. 调用 `builder.finish(bounds, units_per_em)` 构建完整轮廓
3. 存入缓存后返回

#### `glyph_outline_bounds_in_ems(glyph_id, out_outline) -> Option<Rect<f32>>`
**仅获取轮廓边界**（不保留完整轮廓细节）。先查缓存，有则从缓存取边界和轮廓引用。无缓存则调用 `glyph_outline()` 获取。

#### `with_glyph_raster_image(glyph_id, dpxs_per_em, f) -> Option<R>`
**访问嵌入式位图字形**（如 emoji）。调用 `face.glyph_raster_image()` 获取 `ttf_parser::RasterGlyphImage`，包装为 `GlyphRasterImage` 后传入闭包。

#### `has_glyph_raster_image(glyph_id, dpxs_per_em) -> bool`
检查字形是否包含特定位图尺寸的嵌入式图像。

#### `rasterize_glyph(glyph_id, dpxs_per_em) -> Option<RasterizedGlyph>`
**字形光栅化快捷方式**。委托给 `rasterizer.borrow_mut().rasterize_glyph(self, glyph_id, dpxs_per_em)`。

#### `rasterize_glyph_stable_fallback(glyph_id, dpxs_per_em) -> Option<RasterizedGlyph>`
稳定备选光栅化（跳过 MSDF）。委托给 `rasterizer.borrow_mut().rasterize_glyph_stable_fallback(...)`。

### `Hash / PartialEq for Font`
基于 `FontId` 实现。同一标识符的字体视为相等。

---

## 测试（第 209-261 行）

### `noto_color_emoji_prefers_raster_images`
验证 emoji 字体在光栅化时优先使用色位图路径：
1. 加载 `NotoColorEmoji.ttf`
2. 获取 😀 的 glyph ID
3. 确认 `has_glyph_raster_image()` 返回 true
4. 执行 `rasterize_glyph()`，确认 `atlas_kind == AtlasKind::Color`
