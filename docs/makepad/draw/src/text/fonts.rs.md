# `draw/src/text/fonts.rs` — 字体管理器

## 概述

这是 Makepad 渲染层与字体子系统的桥梁（328 行）。`Fonts` 类型整合了 `Layouter`（布局）、`Rasterizer`（光栅化）、`SlugAtlas`（向量字形缓存）以及 GPU 纹理管理，提供完整的文本渲染 API。它还管理 MSDF 作业的线程调度。

## 核心类型

### `Fonts`（第 32-308 行）
字体的高级渲染管理器。持有：
- `layouter: Layouter` — 布局引擎（内含 Loader、Shaper、Rasterizer）
- `atlas_texture: Texture` — GPU 图集纹理（BGRA u8 格式）
- `slug_atlas: SlugAtlas` — SLUG 向量字形缓存
- `slug_min_dpxs_per_em` — SLUG 启用阈值（大字号才使用 SLUG）
- `slug_new_glyphs_per_redraw` — 每帧最大新增 SLUG 字形数（Linux/Windows 限制为 1，其他平台无限）
- `slug_budget_redraw_id / slug_built_glyphs_this_redraw` — 每帧预算追踪
- `msdf_job_sender / msdf_result_receiver` — MSDF 异步作业通道

#### `Fonts::new(cx, settings) -> Self`
构造方法。启动 MSDF 后台线程：
1. 从 settings 创建 `Layouter`
2. 从 Rasterizer 获取图集尺寸和 MSDF 设置
3. 创建 `FromUI<->ToUI` 通道对
4. 启动后台线程循环：接收 `QueuedMsdfJob` → 调用 `msdfer.outline_to_msdf()` → 发送 `CompletedMsdfJob`

#### `get_or_layout(params) -> Rc<LaidoutText>`
文本布局入口。委托给 `layouter.get_or_layout(params)`。

#### `prepare_textures(cx) -> bool`
**每帧纹理准备工作**。在绘制前调用。流程：
1. 检查图集是否需要重置——若 Rasterizer 请求重置，调用 `on_atlas_reset()` 返回 false
2. 获取并应用已完成的 MSDF 异步作业
3. 将所有待处理 MSDF 作业派发到后台线程
4. 刷新 SLUG 纹理（上传新增曲线/波段数据）
5. 更新图集纹理（`prepare_atlas_texture`），将脏矩形区域上传到 GPU
6. 返回 true 表示纹理已准备好

#### `prepare_atlas_texture(cx)`
将 Rasterizer 图集的脏矩形上传到 GPU 纹理。流程：
1. 获取脏矩形范围
2. 将图集像素转换为 `Vec<u32>`（BGRA → u32 的零拷贝 reinterpret）
3. 调用 `atlas_texture.put_back_vec_u32()` 上传到 GPU

#### `prepare_atlases_if_needed(cx)`
在帧结束前回读 GPU 纹理数据到 CPU 图集（用于下次帧的脏矩形累积）。双向纹理同步机制。

#### `get_or_cache_slug_glyph(redraw_id, font, glyph_id) -> SlugGlyphCacheResult`
**SLUG 字形缓存**。流程：
1. 调用 `slug_atlas.get_or_cache_glyph(font, glyph_id, false)` 尝试不构建地获取
2. 若返回 Deferred（未缓存且 not build）：
   a. 检查每帧预算：若当前帧已构建数达到上限，再次返回 Deferred
   b. 否则调用 `slug_atlas.get_or_cache_glyph(font, glyph_id, true)` 实际构建
3. 返回结果（Ready / NeedsUpload / Deferred / Unavailable）

#### `dispatch_msdf_jobs()`
从 Rasterizer 取出所有待处理 MSDF 作业，通过 `FromUISender` 发送到后台线程。

#### `apply_completed_msdf_jobs() -> usize`
轮询接收已完成的 MSDF 作业，调用 `rasterizer.apply_completed_msdf_job()` 应用每个结果，返回计数。

### MSDF 线程架构
```
UI 线程                                    后台线程
  │                                          │
  ├─ dispatch_msdf_jobs()                    │
  │    → FromUISender.send(job) ──────────── │
  │                                          ├─ worker_rx.recv()
  │                                          │   msdfer.outline_to_msdf()
  │                                          ├─ worker_tx.send(completed)
  │    ← ToUIReceiver.try_recv() ────────────│
  ├─ apply_completed_msdf_job()              │
  │                                          │
  └─ (MSDF 像素写入图集)                     │
```

### 辅助函数

#### `bgra_vec_into_u32(vec: Vec<Bgra>) -> Vec<u32>`
零成本 reinterpret 转换。`Bgra` 是 `#[repr(transparent)]` 包装的 `u32`，因此内存布局完全一致。用 `ManuallyDrop` 防止双重释放，然后 `Vec::from_raw_parts` 转换元素类型。

#### `u32_vec_into_bgra(vec: Vec<u32>) -> Vec<Bgra>`
反向转换。
