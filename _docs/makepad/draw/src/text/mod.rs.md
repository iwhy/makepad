# `draw/src/text/mod.rs` — 文本子系统模块入口

## 概述

这是 Makepad 文本渲染子系统的模块根文件。它通过 `pub mod` 声明导出了全部 22 个子模块，构成了完整的文本渲染管线。

## 模块清单

| 模块 | 用途 |
|------|------|
| `color` | 颜色类型定义 |
| `font` | 核心字体抽象（Font, FontId, GlyphId） |
| `font_atlas` | 字形图集（FontAtlas 在线矩形打包） |
| `font_face` | 字体表面抽象（封装 ttf_parser/rustybuzz Face） |
| `font_family` | 字体系列抽象（多字体回退链） |
| `fonts` | 字体管理器（整合 Layouter/纹理上传/MSDF 线程调度） |
| `geom` | 几何类型（Point, Rect, Size） |
| `glyph_outline` | 字形轮廓提取与 Builder |
| `glyph_raster_image` | 嵌入位图字形（如 emoji）封装 |
| `image` | 像素图操作（Image, Subimage） |
| `intern` | 字符串驻留（Intern） |
| `layouter` | 文本布局引擎（断行、对齐、省略号） |
| `loader` | 字体定义与懒加载缓存 |
| `msdfer` | 多通道有符号距离场生成 |
| `num` | 数值工具（Zero trait） |
| `rasterizer` | 字形光栅化（SDF/MSDF/位图 + 图集管理） |
| `sdfer` | 单通道有符号距离场生成 |
| `selection` | 文本选择与光标模型 |
| `shaper` | HarfBuzz 文本 shaping（字体回退、BiDi） |
| `slice` | 切片工具（group_by 等） |
| `slug_atlas` | SLUG 向量字形缓存（GPU 曲线渲染） |
| `substr` | 子串引用类型（Substr） |

## 数据流

整个文本渲染管线的核心流程如下：

```
Text (Unicode 字符串)
  │
  ▼
Shaper (rustybuzz/HarfBuzz shaping)
  │  字体回退、BiDi 双向文本分解、OpenType features
  │  输出: ShapedText (已 shaping 的字形序列)
  ▼
Layouter (文本布局)
  │  断行 (按词/按字素)、对齐、缩进、省略号截断
  │  输出: LaidoutText (已布局的行 + 字形)
  ▼
Rasterizer (字形光栅化)
  │  SDF/MSDF 生成、色位图解码、图集矩形打包
  │  输出: RasterizedGlyph (图集槽位信息)
  ▼
FontAtlas (图集纹理)
  │  BGRA 像素数据 + 脏矩形追踪
  │  输出: GPU 纹理上传
  ▼
DrawGlyph (GPU 绘制)
  │  shader 读取图集纹理 → 屏幕渲染
```

对于大型/高质量渲染，还有 **SLUG 路径**：

```
LaidoutGlyph
  │  (对于大字号)
  ▼
SlugAtlas (向量字形缓存)
  │  曲线 → 归一化二次贝塞尔 → 上传到 RGBAf32 纹理
  │  输出: GPU 曲线数据 → shader 实时扫描
```
