# `draw/src/text/slug_atlas.rs` — SLUG 向量字形缓存

## 概述

这是 `SlugAtlas` 类型（777 行），实现了一种完全不同的字形渲染路径——**SLUG**（Scalable Large-scale Unified Glyphs）。与传统的 SDF/MSDF 位图图集不同，SLUG 将字形轮廓曲线数据直接上传到 GPU 纹理，由 shader 在绘制时实时计算覆盖率。这使得超大字号渲染没有质量损失（无缩放锯齿），且纹理尺寸与字形复杂度成正比而非与字号成正比。

### SLUG 数据流
```
字形轮廓 (GlyphOutline)
  │
  ▼
outline_to_normalized_quads()
  │  三次贝塞尔 → 二次贝塞尔细分
  │  坐标为 em 单位 → 边界框归一化 [0,1]
  │  Y 轴翻转（字体坐标 → 屏幕坐标）
  ▼
Vec<QuadCurve> 存储为 f32 数组
  │  每条曲线 8 个 float (p0.x/y, p1.x/y, p2.x/y, 0, 0)
  ▼
curve_data: Vec<f32> → curve_texture: RGBAf32
  │  追加到 append-only 纹理
  ▼
GPU Shader: 对每个像素遍历所有曲线
  │  实时扫描转化 → 覆盖率 → alpha
```

## 核心类型

### `SlugAtlas`（第 51-312 行）
SLUG 图集管理器。持有：
- `curve_data / band_data` — 曲线和波段数据的 flat float 数组
- `curve_texture / band_texture` — GPU RGBAf32 纹理
- `curve_dirty / band_dirty` — 数据变更标记
- `curve_uploaded_floats / band_uploaded_floats` — 已上传的 float 数量
- `cache_generation / uploaded_generation` — 缓存世代计数
- `cached_glyphs: FxHashMap<SlugGlyphKey, CachedSlugGlyphInfo>` — 缓存字形
- `missing_glyphs: FxHashSet<SlugGlyphKey>` — 已知缺失的字形（免重复尝试）

#### `SlugAtlas::new(cx) -> Self`
构造方法。初始化曲线/波段数据为空，创建 1x1 RGBAf32 纹理作为初始占位，所有计数为零。

#### `get_or_cache_glyph(font, glyph_id, can_build) -> SlugGlyphCacheResult`
**SLUG 字形获取/构建入口**。策略：
1. 查 `cached_glyphs`：
   - 若已缓存且 `generation <= uploaded_generation`（已上传）→ `Ready(info)`
   - 若已缓存但未上传 → `NeedsUpload { generation, glyph }`
2. 查 `missing_glyphs` → `Unavailable`
3. 若 `!can_build` → `Deferred`（当前帧不构建，可能后续帧再尝试）
4. 构建字形：
   a. `font.glyph_outline(glyph_id)` 获取轮廓
   b. `build_glyph()` 将轮廓转换为归一化二次曲线
   c. 成功则缓存并返回 `NeedsUpload`，失败则加入 `missing_glyphs`

#### `prepare_textures(cx) -> bool`
将新增的曲线/波段数据上传到 GPU 纹理。使用 `prepare_append_only_rgba_f32_texture()` 方法逐纹理处理。上传成功后更新 `uploaded_generation`。

#### `prepare_append_only_rgba_f32_texture(...) -> bool`
**只追加纹理上传**（第 191-250 行）。高效处理增量数据追加：
1. 计算新纹理尺寸（固定宽度 2048，高度按数据量向上取整）
2. 检查是否需调整纹理尺寸
3. 复制新数据到纹理缓冲区（仅新增部分，避免全量重传）
4. 设置 `TextureUpdated`（Full 或 Partial dirty rect）
5. 更新已上传计数

#### `appended_dirty_rect(...) -> Option<RectUsize>`
计算追加数据的脏矩形。当新增数据在当前行内时只返回该行片段，跨行时返回整行范围。

#### `build_glyph(font, outline) -> Option<SlugGlyphInfo>`
**构建单个 SLUG 字形**。流程：
1. 获取轮廓边界，检查是否有效（宽高 > 1e-6）
2. 调用 `outline_to_normalized_quads()` 将轮廓转换为归一化二次贝塞尔曲线
3. 将曲线追加到 `curve_data`（每条曲线 8 个 float）
4. 标记 `curve_dirty`，递增 `cache_generation`
5. 返回 `SlugGlyphInfo`（边界、曲线偏移/计数、波段偏移/计数）

### `SlugGlyphInfo`（第 23-32 行）
SLUG 字形信息。包含：边界框（em 单位）、在曲线/波段数据中的偏移和计数、填充标志。

### `SlugGlyphCacheResult`（第 40-49 行）
缓存查询结果枚举：
- `Ready(info)` — 已上传 GPU，可直接使用
- `NeedsUpload { generation, glyph }` — 已构建但需上传
- `Deferred` — 已缓存但暂不构建（预算限制）
- `Unavailable` — 字形不可用（无轮廓）

---

### `QuadCurve` / `P2`（第 314-325 行）
二维点和二次贝塞尔曲线的简单数据结构。

### `outline_to_normalized_quads(outline, bounds, units_per_em) -> Vec<QuadCurve>`（第 327-399 行）
**将字体轮廓转换为归一化二次贝塞尔曲线**。核心转换：
1. 所有坐标从字体单位转换为 em 单位（除以 `units_per_em`）
2. 再归一化到 `[0, 1]` 范围（相对于边界框宽高）
3. Command 转换：
   - `MoveTo` → 设置当前点
   - `LineTo` → 构造退化的二次曲线（控制点为中点）
   - `QuadTo` → 直接转换为二次曲线
   - `CurveTo` → 递归细分（`cubic_to_quads_recursive`）
   - `Close` → 若当前点 != 轮廓起点，用退化曲线闭合
4. Y 轴翻转：字体坐标 Y 轴向上 → 屏幕坐标 Y 轴向下

### `cubic_to_quads_recursive`（第 458-503 行）
**三次贝塞尔 → 二次贝塞尔递归细分**。算法：
1. 用 `cubic_to_quad_control()` 计算近似二次曲线的控制点
2. 在 t = 0.25, 0.5, 0.75 处采样误差（原三次曲线 vs 近似二次曲线）
3. 若最大误差 <= `CUBIC_TO_QUAD_TOLERANCE`（0.05）或达到最大递归深度（12），接受近似
4. 否则，用 de Casteljau 算法在 t = 0.5 处分裂为两段下半三次曲线，递归处理

---

## 测试（第 505-777 行）

测试覆盖：
1. `appended_dirty_rect`：脏矩形计算的正确性（同行内 vs 跨行）
2. `builds_slug_glyphs_for_uizoo_demo_letters`：验证 A/g/W/S/L 等字形的曲线生成
3. `text_slug_glyphs_use_full_curve_scan`：确认文本渲染强制使用全曲线扫描路径（`band_count == 0`）
4. `uizoo_demo_letters_produce_nonzero_slug_coverage`：验证 SLUG 覆盖率的参考实现——用 CPU 模拟 SLUG shader 的扫描转换算法，确认 `max_alpha > 0.2`
