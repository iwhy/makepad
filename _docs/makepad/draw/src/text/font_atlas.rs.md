# `draw/src/text/font_atlas.rs` — 字形图集管理

## 概述

这是 `FontAtlas<T>` 泛型类型（307 行），实现了字形的在线纹理图集分配与管理。它是一个泛型容器，被实例化为三种具体类型：
- `GrayscaleAtlas = FontAtlas<Bgra>` — 灰度 SDF 图集
- `ColorAtlas = FontAtlas<Bgra>` — 彩色位图图集
- `MsdfAtlas = FontAtlas<Bgra>` — MSDF 图集

所有三种都使用 `Bgra` 像素类型（`#[repr(transparent)]` 包装 `u32`），但在逻辑上承载不同的渲染数据。

## 核心类型

### `FontAtlas<T>`（第 12-188 行）
泛型图集容器。持有：
- `needs_reset: bool` — 空间耗尽时设为 true，请求外部重置
- `image: Image<T>` — 像素图像数据
- `dirty_rect: Rect<usize>` — 脏矩形累积区域
- `free_rects: Vec<Rect<usize>>` — 在线打包器的自由矩形列表
- `cached_glyph_image_rects: FxHashMap<GlyphImageKey, Rect<usize>>` — 字形 → 图集位置映射

#### `FontAtlas::new(size) -> Self`
构造指定尺寸的图集。初始化图像为全零，自由矩形列表为 `[Rect::from(size)]`（整个图集为一大块自由区域）。

#### `request_reset()`
标记需要重置。当 `allocate_glyph_image` 发现图集无空间时会调用。

#### `reset_if_needed() -> bool`
若需要重置：清空脏矩形、重置自由矩形列表、清空 glyph 位置缓存、清除重置标记。返回 true 表示发生了重置。

#### `get_or_allocate_glyph_image(key) -> Option<GlyphImage<'_, T>>`
**字形图集分配入口**。优先返回已缓存的 `GlyphImage::Cached(rect)`，否则调用 `allocate_glyph_image` 分配新区域并返回 `GlyphImage::Allocated(subimage)`。

#### `allocate_glyph_image(size) -> Option<Rect<usize>>`
分配指定尺寸的区域。调用 `place_rect(size)` 在线打包，成功则标记脏矩形，失败则设置 `needs_reset`。

#### `place_rect(size) -> Option<Rect<usize>>`
**在线 max-rects 矩形打包**。算法：
1. 遍历 `free_rects`，筛选能容纳 `size` 的自由矩形
2. 按 `(short_side, long_side, area)` 三元组排序，取最短边优先的布置
3. 调用 `split_free_rects(placed)` 将放置区域周围的自由矩形分裂
4. 调用 `prune_free_rects()` 移除被包含的冗余矩形
5. 返回放置位置

#### `split_free_rects(used)`
十字形分裂：每个与 `used` 相交的自由矩形被替换为 2-4 个不相交的子矩形（上、下、左、右剩余区域）。

#### `prune_free_rects()`
移除被其他自由矩形完全包含的冗余矩形，保持自由矩形列表最小化。双重循环检查所有配对。

#### `get_cached_glyph_image_mut(rect) -> SubimageMut<'_, T>`
获取已分配区域的可变子图像引用，并标记该区域为脏（用于后续纹理上传）。

#### `mark_dirty_rect(rect)`
累积脏矩形：若已有脏矩形则求并集，否则设为该矩形。

#### `take_dirty_image() -> Subimage<'_, T>`
取出脏矩形子图像引用，同时将内部脏矩形重置为零（消费模式）。

### `GlyphImageKey`（第 288-294 行）
字形图集键。包含：`font_id`（字体标识）、`glyph_id`（字形标识）、`size`（渲染尺寸）、`kind`（类型：SDF/MSDF/Color）。实现 `Eq + Hash`。

### `GlyphImageKind`（第 296-301 行）
字形图像类型枚举：`OutlineSdf`（灰度 SDF）、`OutlineMsdf`（多通道 SDF）、`Color`（彩色位图）。

### `GlyphImage<'a, T>`（第 303-307 行）
图集查询结果枚举：`Cached(Rect<usize>)` 表示已存在（返回位置），`Allocated(SubimageMut)` 表示新分配（返回可写入的子图像引用）。

### `rects_intersect` / `split_free_rect` / `push_non_empty_rect`
辅助函数，与 `rasterizer.rs` 中的对应函数逻辑一致。
