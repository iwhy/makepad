# `draw_text.rs` — 文本着色器（DrawText）

本文件实现了 Makepad 完整的文本渲染管道，包括字体样式管理、文本排版、glyph 渲染（三种模式）、以及 Slug/UVE 等中间数据结构。是 draw crate 中最复杂的着色器之一（约 85KB）。

## `mod.draw.DrawText` — 文本着色器

### 注册

```rust
mod.draw.DrawText = mod.std.set_type_default() do #(DrawText::script_shader(vm)){...}
```

### Fragment Shader — `fn pixel()`

#### 核心 SDF 渲染逻辑

1. **从 three_band 纹理采样字形 SDF**：将当前 UV 坐标采样到 `three_band.r`（主 SDF 距离场），获得带符号距离值
2. **抗锯齿转换**：使用 `smoothstep(0.5 - aa_width, 0.5 + aa_width, three_band.r)` 将连续 SDF 值转换为硬边缘或平滑边缘的覆盖度
3. **颜色混合**：覆盖度乘以 `varying_color`，支持文字颜色、透明度
4. **从属颜色输出**：输出颜色后，将 `color * coverage` 写入从属颜色缓冲区以支持文字上方的二次编辑

#### 三种 Glyph 渲染模式

**模式 1 — Slug Glyph（预渲染字形 Slug）**：
- 使用 `glyph_texture` (ArrayTexture2D) 采样预渲染的灰度 glyph
- `glyph_texture_size` 包含纹理尺寸和归一化缩放
- `varying_glyph_texture_index` 选择纹理数组层
- 双线性过滤采样，直接读出灰度 alpha 值
- 主要用于缓存加速已渲染的字形

**模式 2 — UVE Glyph（基于 UV 展开）**：
- 使用 `glyph_texture_uv` 编码的 UV 坐标在字体纹理图集中采样
- `varying_uv_bias` 提供 UV 偏移和缩放
- 支持 subpixel 定位（通过 `varying_uv_bias` 的分数部分）
- 适用于普通文本渲染，在字体内存和渲染质量之间取得平衡

**模式 3 — SDF Three-Band Glyph（有符号距离场，三频带）**：
- 使用 `three_band` 纹理（可能是 RGB 三个通道的纹理）存储三频带 SDF
- 每个频带存储距离范围的不同分辨率细节
- 高频带提供边缘细节，低频带提供大的距离信息
- 三频带组合可在大幅度放大时保持边缘清晰，同时避免传统 SDF 的锯齿
- 主要用于可变字体大小的高质量渲染

### 字体纹理图集管理

- 所有 glyph 的渲染图像存储在共享纹理图集中
- 图集动态扩展，新 glyph 按需渲染
- 纹理图集更新通过 `UniformTexture` 系统同步到 GPU

### Slug 数据结构

Slug 是 `DrawText` 的中间渲染产物，代表一段已排版的文本行（包括每个字符的位置、字形索引、字体样式等）。具体字段：

- **`glyph_index`**：字符在字体中的索引
- **`glyph_texture_index`**：在 glyph 纹理数组中的层索引
- **`color`**：字符颜色
- **`pos`**：字符在行中的位置 (x, y)
- **`scale`**：字符缩放
- **`font_size`**：字体大小控制
- **`bounds`**：字符边界框
- **`underline_offset`**、**`strikethrough_offset`**：装饰线位置
- **`lsb_delta`**、**`rsb_delta`**：左右侧边距微调
- **`is_space`**、**`is_tab`**：空白字符标志
- **`text_style`**：指向 TextStyle 的引用
- **`has_color_glyph`**：彩色表情符号/字形标志
- **`has_thin_lines`**：细线优化标志
- **`is_emoji`**：emoji 标志
- **`is_variable`**：可变字体标志

## `mod.draw.TextStyle` — 文本样式

### 注册

```rust
mod.draw.TextStyle = mod.std.set_type_default() do #(TextStyle::script_component(vm)){...}
```

### 字段（35 个）

- **字体控制**：`font_size`（字号）、`line_height`（行高）、`letter_spacing`（字距）、`word_spacing`（词距）
- **对齐**：`text_align`（水平左/中/右）、`vertical_align`（垂直上/中/下）
- **装饰**：`underline`、`strikethrough`、`overline`（装饰线启用/颜色/宽度/偏移）
- **样式**：`italic`（斜体）、`small_caps`（小型大写）
- **颜色**：`color`、`selection_color`、`cursor_color` 等
- **排版**：`tab_size`、`indent`、`wrap`（换行模式）、`max_lines`、`overflow` 等
- **间距**：`padding`、`margin`
- **高级**：`variable_font_settings`、`font_feature_settings`、`font_variation_settings`
- **布局**：`vertical_scroll`、`horizontal_scroll`、`first_line_offset`、`last_line_offset`

### 功能

- 支持从 URL 加载字体（`font_url` 字段）
- 支持字体回退链（`font_fallback_list`）
- 支持可变字体轴设置（`font_variation_settings`）
- 排版控制（行高、字距、对齐、缩进等）
- 支持文本选择高亮（`selection_color`、`selection_background`）

### 字体加载机制

- `TextStyle` 通过字体管理系统按名称/URL 查找字体
- 支持多字体回退：首选字体缺失 glyph 时自动尝试回退字体
- 可变字体通过 `font_variation_settings` 控制字重、宽度等轴

## `mod.draw.FontFamily` — 字体系列

### 注册

```rust
mod.draw.FontFamily = mod.std.set_type_default() do #(FontFamily::script_component(vm)){...}
```

存储字体系列名称、字体文件路径、回退链。支持系统字体和自定义字体。

## RenderAttrs

渲染属性枚举，控制文本的基础渲染方式：
- `FontSlug`：使用 slug glyph（预渲染轮廓）
- `FontUVE`：使用 UVE 纹理
- `FontThreeBand`：使用三频带 SDF
- `NotImplemented`：未实现

## `mod.draw.FontGlyph` / `mod.draw.FontGlyphSet`

字体字形管理：
- `FontGlyph`：单个字形（glyph_index、轮廓路径、bounds、advance 等）
- `FontGlyphSet`：字形集合，以 glyph index 为 key 的哈希表

## `mod.draw.FontTextureInfoArray` — 字体纹理信息数组

维护 glyph 纹理图集的元数据数组：
- 每个 glyph 在纹理图集中的位置 `(u, v, w, h)`
- 纹理索引
- 缩放信息

## `mod.draw.GlyphTextureId` — 字形纹理 ID 系统

将逻辑 glyph 映射到物理纹理位置：
- `id`：glyph_id
- `texture_index`：纹理数组层索引
- `uv`：纹理中的 UV 坐标
- `size`：glyph 尺寸

## 文本渲染管线总结

1. **排版阶段**（CPU）：
   - TextStyle 确定字体、字号、对齐等参数
   - HarfBuzz/Bidi 库进行双向文本分析和字形选择
   - 计算每个字形的位置、尺寸、边框

2. **Slug 生成阶段**（CPU/GPU 混合）：
   - 将排版的文本行组织为 Slug 数据结构
   - 分配 glyph texture 索引
   - 确定渲染模式（Slug/UVE/ThreeBand）

3. **渲染阶段**（GPU）：
   - 顶点着色器传递 slug 参数到片段着色器
   - 片段着色器根据模式采样不同纹理
   - SDF 技术实现高质量抗锯齿
   - 颜色输出到帧缓冲区

4. **特殊渲染**：
   - 下划线、删除线在 slug 渲染后叠加
   - 文本选择高亮在文本后渲染
   - 光标闪烁通过独立的 draw call 实现
