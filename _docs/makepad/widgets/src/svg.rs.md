# svg.rs — SVG 矢量图形渲染组件

## 概述
`Svg` 组件使用 NanoVG/nanosvg 解析和渲染 SVG 文件。支持 SVG 文件的加载、缓存、缩放控制（`fit`/`fill` 模式）和颜色覆盖（`tint`/`colorize`）。通过 `DrawSvg` 绘制器利用 Makepad 的 SDF 管线实现矢量图形的高质量渲染。

## 核心结构

### Svg
- **`draw_svg: DrawSvg`**：SVG 绘制器。
- **`svg_data: Option<ScriptHandleRef>`**：SVG 数据的资源句柄。
- **`image_based: bool`**：是否基于 SVG 的 PNG 光栅化版本（替代即时渲染）。

### DrawSvg
继承自 `DrawVector` 的可配置 SVG 绘制器，包含 `tint`（着色色值）和 `colorize`（是否对 SVG 重新着色 — 对单色图标有用）。

### DrawVector
继承自 `DrawQuad` 的基础矢量绘制类型，使用 NanoVG 渲染管线，包含 SVG 数据句柄、DPI 设置、缩放模式和尺寸信息。

### VectorCache
全局缓存映射 `ScriptHandleRef → VectorCacheValue`，缓存包括 `svg_data`（nano_svg 解析的 SVG）、`nvg_handle`（NanoVG 光栅句柄）和引用计数。`try_gc` 实现垃圾回收。

### VectorCacheValue
缓存条目，包含可选的 svg_data（未光栅化的 SVG）/nvg_handle（NanoVG 光栅化句柄）/image_data（备选 PNG 位图），以及 `ref_count` 引用计数和路径名。

## 核心方法

### Widget 实现

**`draw_walk`**：如果 `svg_data` 存在且没有对应的图像/矢量缓存，先通过 `cx.get_script_handle_data_path` 加载 SVG 文件路径，再调用 `cache_setup_svg_ref` 在 VectorCache 中创建或增加引用计数。然后委托 DrawVector 绘制。

**`handle_event`**：处理 SVG 数据的资源加载完成事件（`ScriptHandleLoaded`），关联句柄和数据。

### DrawSvg 绘制

**`DrawVector::draw_abs`/`draw_walk`**：在绝对坐标或布局坐标下绘制矢量图形。检查 VectorCache，根据 `mode`（`Fit`/`Fill`）计算缩放和居中偏移。`Fit` 模式保持宽高比使 SVG 完全可见；`Fill` 模式保持宽高比完全填充区域。通过 `cx2d.nvg()` 调用 NanoVG 渲染。

### SVG 数据管理

**`set_svg_data`**：设置 SVG 数据句柄。先清理旧句柄的缓存（递减引用计数），再加载新 SVG 文件到 VectorCache，最后设置 `draw_svg.svg_data = svg_data` 并触发重绘。

**`cache_setup_svg_ref`**：从文件路径创建 `ScriptHandleRef`，初始化 VectorCacheEntry（解析 SVG 数据、可选 PNG 备选回退），设置 `draw_svg.svg_data` 和 `draw_svg.svg_size`。

**`cache_unref_svg`**：递减缓存引用计数，引用归零时移除缓存条目。

### 颜色控制

**`set_tint`/`set_colorize`**：设置覆盖色和着色开关。

### VectorCache

**`new`**：创建容量为 64 的 LRU 缓存。

**`cache_setup_svg_ref`**：从文件路径查找或创建缓存条目。支持 SVG（首选）和 PNG 回退（当 SVG 解析失败时）。对 Chrome DevTools 的 `image/svg+xml` 数据 URI 也提供支持。

**`cache_unref`**：释放引用，引用计数归零时标记为可回收。

**`try_gc`**：垃圾回收 — 清理所有引用计数为 0 的条目，并释放 NanoVG 句柄。
