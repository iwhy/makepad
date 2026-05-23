# `draw/src/text/rasterizer.rs` — 字形光栅化与图集管理

## 概述

这是 Makepad 的字形光栅化引擎（905 行），负责将矢量字形轮廓转换为像素数据并存储在 GPU 图集中。支持三种光栅化路径：
- **SDF**（单通道有符号距离场）：传统字体渲染，适合小字号
- **MSDF**（多通道有符号距离场）：高质量放大渲染，无锯齿
- **Color**（嵌入式位图）：emoji 等彩色位图字形

## 核心类型

### `Rasterizer`（第 16-436 行）
光栅化器的中央状态机。持有：
- `sdfer: Sdfer` / `msdfer: Msdfer` — 距离场生成器实例
- `msdf_resolution: MsdfResolutionSettings` — MSDF 分辨率策略
- `msdf_complexity: MsdfComplexitySettings` — MSDF 复杂度阈值
- `outline_rasterization_mode` — 当前模式（Sdf 或 Msdf）
- `atlas: ColorAtlas` — BGRA 四通道图集（实际为 `FontAtlas<Bgra>`）
- `allocator: MultiPlaneAllocator` — 四平面矩形打包器
- `cached_slots: FxHashMap<GlyphImageKey, AtlasSlot>` — 字形 → 图集槽位映射
- `outline_msdf_ready / outline_msdf_pending / outline_msdf_failed` — MSDF 异步状态跟踪
- `queued_msdf_jobs` — 待发送给后台线程的 MSDF 作业队列
- `atlas_epoch` — 图集版本号（用于丢弃过期异步结果）

#### `Rasterizer::new(settings: Settings) -> Self`
构造光栅化器。初始化 SDF/MSDF 引擎、图集、四平面分配器，所有缓存集合为空。

#### `on_atlas_reset()`
图集重置时调用。清空分配器、所有缓存状态，递增 `atlas_epoch`。图集重置发生在 `allocate_sdf_slot` 或 `allocate_shared_slot` 图集空间耗尽时。

#### `take_queued_msdf_jobs() -> Vec<QueuedMsdfJob>`
取出所有待处理的 MSDF 作业，移交 `Fonts::dispatch_msdf_jobs()` 发送给后台线程。

#### `apply_completed_msdf_job(job: CompletedMsdfJob)`
应用后台线程完成的 MSDF 结果。安全检查：
1. `job.epoch == self.atlas_epoch` — 忽略过期结果（图集已重置）
2. `self.outline_msdf_pending.contains(&job.key)` — 确认作业确实在等待中
3. `self.cached_slots.contains(&job.key)` — 槽位仍有效
4. 像素数据长度匹配 key 中的尺寸

成功后将 MSDF 的 RGB 通道写入图集对应区域，保留 alpha 通道（来自已播种的 SDF 覆盖，确保 MSDF 计算完成前有稳定视觉效果）。然后将 key 移入 `outline_msdf_ready`。

#### `allocate_sdf_slot(key) -> Option<(AtlasSlot, bool)>`
分配 SDF 槽位。优先返回已缓存的槽位（`(slot, false)`），否则从 `MultiPlaneAllocator` 分配新槽位（`(slot, true)`）。分配失败时请求图集重置。

#### `allocate_shared_slot(key) -> Option<(AtlasSlot, bool)>`
分配 MSDF 共享槽位（需要在四个 BGRA 平面的相同位置都有可用空间）。与 `allocate_sdf_slot` 逻辑类似但使用 `allocate_shared_slot()` 方法。

#### `seed_msdf_slot_from_sdf(slot, sdf_glyph)`
在异步 MSDF 完成之前，从已有的 SDF 灰度图集复制像素到 MSDF 槽位作为临时覆盖。RGBA 四个通道都填充 SDF 值，保证视觉上可以接受。

#### `rasterize_glyph(font, glyph_id, dpxs_per_em) -> Option<RasterizedGlyph>`
**字形光栅化的顶级入口**。优先级策略：
1. 优先检查嵌入式位图（`rasterize_glyph_raster_image`）—— 用于 emoji 等彩色字形
2. 若无位图，使用轮廓光栅化（`rasterize_glyph_outline`）
3. 轮廓光栅化的模式由 `outline_rasterization_mode` 决定

#### `rasterize_glyph_stable_fallback(font, glyph_id, dpxs_per_em) -> Option<RasterizedGlyph>`
备选光栅化路径：位图 → SDF（跳过 MSDF）。用于需要绝对稳定渲染结果的场景。

#### `rasterize_glyph_outline(font, glyph_id, dpxs_per_em) -> Option<RasterizedGlyph>`
按 `outline_rasterization_mode` 分发：
- `Sdf` → `rasterize_glyph_outline_sdf()`
- `Msdf` → `rasterize_glyph_outline_msdf()`

#### `rasterize_glyph_outline_sdf(font, glyph_id, dpxs_per_em) -> Option<RasterizedGlyph>`
**SDF 光栅化实现**。完整流程：
1. 字型分辨率不低于 `min_dpxs_per_em`
2. 提取字形轮廓（`font.glyph_outline_bounds_in_ems` + `glyph_outline`）
3. 计算图集图像尺寸（`glyph_outline_image_size` + padding）
4. 构建 `GlyphImageKey`，尝试 `allocate_sdf_slot`
5. 若为新分配的槽位：
   a. 创建 `Image<R>` 进行覆盖率光栅化（`outline.rasterize()`）
   b. 调用 `sdfer.coverage_to_sdf()` 将覆盖率转换为 SDF
   c. 将 SDF 像素写入图集对应平面
6. 返回 `RasterizedGlyph`（平面选择由分配器 Round-Robin 决定）

#### `rasterize_glyph_outline_msdf(font, glyph_id, dpxs_per_em) -> Option<RasterizedGlyph>`
**MSDF 光栅化实现**（异步路径）。完整流程：
1. 小字号（`<= min_request_dpxs_per_em`）直接降级为 SDF
2. 估算轮廓复杂度（`estimate_outline_complexity`），若超出阈值降级为 SDF
3. 检查 `outline_msdf_failed` 集（曾经失败的 key 直接降级）
4. 检查 `outline_msdf_ready` 集（异步已完成，返回 MSDF 槽位）
5. **关键路径**：先光栅化 SDF 作为即时显示，同时分配共享 MSDF 槽位并从 SDF 播种，然后将 MSDF 作业入队（后台线程执行 `msdfer.outline_to_msdf()`）
6. 返回 SDF 版本的 glyph（临时显示），MSDF 完成后自动替换

#### `rasterize_glyph_raster_image(...) -> Option<RasterizedGlyph>`
**嵌入式位图光栅化**（如 emoji）。流程：
1. 调用 `font.with_glyph_raster_image()`，获取 `GlyphRasterImage`
2. 构建 key + `allocate_shared_slot`
3. 解码位图数据到图集子区域（带 2 像素 padding）
4. 返回 `AtlasKind::Color` 的 `RasterizedGlyph`

---

### `AtlasSlot` / `AtlasPlane` 系统（第 438-488 行）
图集槽位包含矩形区域和平面索引（R/G/B/A）。`AtlasPlane` 提供 `set()` 和 `get()` 方法操作 BGRA 像素的特定通道。

---

### `MultiPlaneAllocator`（第 491-581 行）
四平面矩形打包器。四个独立的 `RectPacker` 实例对应 BGRA 四个通道。

#### `allocate_sdf_slot(size) -> Option<AtlasSlot>`
SDF 槽位分配算法：
1. 遍历所有四个平面，对每个平面调用 `peek_best_fit()` 获取最佳适配评分
2. 评分策略：`(plane_used_area, plane_offset, short_side, long_side, area)` — 先看负载均衡（`plane_used_area`），再看 Round-Robin 偏移，最后看空间效率
3. 分配后更新 `plane_used_area`，旋转 Round-Robin 游标

#### `allocate_shared_slot(size) -> Option<Rect<usize>>`
MSDF 共享槽位需要在四个平面都有相同区域可用。算法：
1. 从平面 0 的自由矩形候选开始
2. 逐个与平面 1/2/3 的自由矩形求交集（`rect_intersection`），筛选出所有平面共有的可用区域
3. 裁剪包含矩形（`prune_contained_rects`）
4. 用 `choose_best_fit_origin` 在共有区域中选择最优布置原点
5. 在四个平面上同时 `reserve()` 该区域

---

### `RectPacker`（第 591-672 行）
在线矩形打包器（max-rects 算法）。维护自由矩形列表，新矩形放入最优适配位置后分裂自由区域。
- `peek_best_fit(size)`：返回最佳适配评分（短边优先、长边次之、面积第三）
- `allocate(size)`：分配新矩形
- `reserve(used)`：预留区域，将相交的自由矩形分裂为剩余子矩形

---

### 辅助函数（第 674-792 行）
- `choose_best_fit_origin`：从候选矩形中选择最优布置点（短边优先）
- `prune_contained_rects`：移除被其他矩形包含的冗余自由矩形
- `rect_intersection` / `rects_intersect`：矩形求交/相交检测
- `split_free_rect`：自由矩形分配后按十字形分裂为 2-4 个子矩形（上下左右剩余区域）

---

### `RasterizedGlyph`（第 836-845 行）
光栅化输出结构。包含：图集类型（Grayscale/Color/Msdf）、图集尺寸、在图集中的边界、padding、平面索引、原位偏移（design space → pixel space）、dpxs_per_em。

### `RasterizationMode` / 复杂度估计
- `estimate_outline_complexity`：估算轮廓的渲染复杂度（直线=1 段、二次曲线=8 段、三次曲线=12 段）
- `is_msdf_complexity_acceptable`：`outline_commands <= max` 且 `estimated_segments <= max` 时接受 MSDF
