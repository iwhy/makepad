# `glyph_raster_image.rs` — 嵌入式位图字形解码

## 文件定位

该文件处理**嵌入式位图字形（Embedded Bitmap / Color Font）**的解码。不同于矢量轮廓字形（由 `glyph_outline.rs` 处理），位图字形以预渲染的 PNG 图像形式嵌入字体文件中（常见于彩色 emoji 字体如 Apple Color Emoji、Segoe UI Emoji 等）。

## 核心结构

### `GlyphRasterImage<'a>`
```rust
pub struct GlyphRasterImage<'a> {
    origin_in_dpxs: Point<f32>,  // 字形原点在像素坐标系中的偏移
    dpxs_per_em: f32,           // 该位图的每 em 像素数（原始渲染分辨率）
    format: Format,             // 编码格式（当前仅支持 PNG）
    data: &'a [u8],             // 原始编码数据（PNG 字节流）
}
```

#### 构造方法
- **`from_raster_glyph_image(image: ttf_parser::RasterGlyphImage)`** — 从 `ttf_parser` 解析出的 `RasterGlyphImage` 构造实例。提取 origin 偏移、pixels_per_em 和原始字节数据。格式通过 `Format::from_raster_image_format()` 进行映射，不支持格式返回 `None`。

#### 查询方法
- **`origin_in_dpxs()`** — 返回字形的像素原点偏移量（x, y），用于在布局中定位位图
- **`size_in_dpxs()`** — 返回位图解码后的尺寸（像素宽度和高度）
- **`bounds_in_dpxs()`** — 返回 `Rect<f32>` 类型的包围盒（含 origin 和 size）
- **`dpxs_per_em()`** — 返回位图原始的每 em 像素密度

### `Format` 枚举
```rust
pub enum Format {
    Png,
}
```

**`from_raster_image_format(format)`** — 将 `ttf_parser::RasterImageFormat` 映射为内部格式。当前仅支持 `PNG` 格式，其他格式均返回 `None`。

## 解码实现

### `decode_size()` / `decode_size_png()`
仅解码 PNG 头部以获取图像尺寸，不解码完整像素数据：
1. 创建 `ZCursor` 包装原始字节
2. 实例化 `PngDecoder` 并调用 `decode_headers()`
3. 提取 `dimensions()` 获取宽高

### `decode(image)`
主解码方法，将 PNG 数据解码为 BGRA 格式的像素图像：

1. **PNG 解码**：使用 `makepad_zune_png` crate 的 `PngDecoder` 完成完整解码
2. **色彩空间适配**：根据 `num_components` 进行通道转换：

   | 组件数 | 原始格式 | 转换方式 |
   |--------|---------|---------|
   | 4 | RGBA | 直接交换 R↔B 通道（RGBA → BGRA） |
   | 3 | RGB | RGB → BGRA，Alpha 设为 255 |
   | 2 | 灰度+Alpha | 灰度值复制到 R/G/B，保留 Alpha |
   | 1 | 灰度 | 灰度值复制到 R/G/B，Alpha 设为 255 |
   | 其他 | 不支持 | 打印警告，跳过 |

3. **像素顺序**：从左上角开始逐行读取，保持图像坐标系一致

## 与 `glyph_outline.rs` 的关系

两者互斥——同一个字形在同一字号下要么由矢量轮廓定义，要么由嵌入式位图定义。`GlyphOutline` 用于可缩放字体（普通文字），`GlyphRasterImage` 用于颜色字体/emoji。渲染管线根据字体表的存在性选择使用哪个：

- 若字体表中有 `CBDT`/`CBLC`（CBDT 彩色位图）或 `sbix` 表 → 使用 `GlyphRasterImage`
- 否则 → 使用 `GlyphOutline` 进行矢量渲染

## 依赖的外部 crate

- **`rustybuzz` / `ttf_parser`** — 提供 `RasterGlyphImage` 类型，是字体解析的上层接口
- **`makepad_zune_png`** — Makepad 自带的 PNG 解码库（基于 `zune-png`），支持标准 PNG 和部分扩展格式
