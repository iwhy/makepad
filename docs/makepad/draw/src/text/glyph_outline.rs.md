# `glyph_outline.rs` — 矢量字形轮廓提取与光栅化

## 文件定位

该文件提供了从 TrueType/OpenType 字体中提取**矢量字形轮廓（Glyph Outline）**的功能，以及将轮廓光栅化为灰度图像的能力。它是 Makepad 文本渲染管线中连接字体解析和 SDF 生成的关键桥梁。

## 核心结构

### `GlyphOutline`
```rust
pub struct GlyphOutline {
    bounds: Rect<f32>,       // 字形在字体坐标系中的包围盒
    units_per_em: f32,       // 字体的 UPM（Units Per Em）
    commands: Vec<Command>,  // 轮廓绘制命令序列
}
```

#### 方法
- **`origin_in_ems()`** — 将包围盒原点从字体单位转换到 em 单位（除以 `units_per_em`）
- **`size_in_ems()`** — 将包围盒尺寸转换到 em 单位
- **`bounds_in_ems()`** — 返回 em 坐标系下的完整包围盒 `Rect`
- **`commands()`** — 返回轮廓命令序列的引用
- **`rasterize_transform(dpxs_per_em)`** — 构造从 em 坐标到像素坐标的变换矩阵：
  1. 平移 `-origin`，将字形原点移至 (0,0)
  2. 均匀缩放 `dpxs_per_em / units_per_em`，将 em 单位映射为像素
- **`rasterize(dpxs_per_em, output)`** — 将轮廓直接光栅化为单通道灰度图像

### `rasterize()` 方法详解

该方法是字形轮廓的**直接光栅化**实现，使用外部 `ab_glyph_rasterizer` crate：

1. 创建与输出图像尺寸匹配的 `Rasterizer` 实例
2. 遍历所有 `Command`，对每个命令：
   - 将端点/控制点通过 `rasterize_transform` 变换到像素坐标
   - 调用对应光栅化方法（`draw_line`、`draw_quad`、`draw_cubic`）
   - `Close` 命令处理：若存在 `last_move`，绘制从当前位置到上次 MoveTo 点的闭合线段
3. 调用 `rasterizer.for_each_pixel_2d()` 遍历每个像素的覆盖值
4. 将覆盖值（浮点）乘以 255 转换为 u8 灰度，并翻转 Y 轴（光栅器坐标原点在左下，输出图像原点在左上）

### `Command` 枚举
```rust
pub enum Command {
    MoveTo(Point<f32>),              // 移动到新子路径起点
    LineTo(Point<f32>),              // 直线段到目标点
    QuadTo(Point<f32>, Point<f32>),  // 二次贝塞尔曲线（控制点，终点）
    CurveTo(Point<f32>, Point<f32>, Point<f32>), // 三次贝塞尔曲线（控制点1，控制点2，终点）
    Close,                           // 闭合当前子路径
}
```

### `Builder`
```rust
pub struct Builder {
    commands: Vec<Command>,
}
```

实现了 `ttf_parser::OutlineBuilder` trait，用于从 `ttf_parser` 的轮廓解析回调中增量构建 `Command` 序列：

- **`move_to(x, y)`** → 推入 `Command::MoveTo`
- **`line_to(x, y)`** → 推入 `Command::LineTo`
- **`quad_to(x1, y1, x, y)`** → 推入 `Command::QuadTo`
- **`curve_to(x1, y1, x2, y2, x, y)`** → 推入 `Command::CurveTo`
- **`close()`** → 推入 `Command::Close`

**`finish(bounds, units_per_em)`** — 消费 Builder，产出完整的 `GlyphOutline` 实例。

## 与 `ttf_parser` 的集成

`Builder` 实现了 `ttf_parser::OutlineBuilder` trait，这个 trait 定义了字体轮廓解析的回调接口。当使用 `ttf_parser` 解析字形轮廓时，它会逐个调用这些回调方法。Makepad 通过实现该 trait 将字体的原始轮廓数据转换为内部的 `Command` 表示，保持了与标准字体解析库的兼容性。

## 数据流

```
ttf_parser 字形数据
    → Builder（实现 OutlineBuilder trait）
    → Vec<Command>
    → GlyphOutline（含包围盒和 UPM）
    → rasterize() → 灰度像素图像
    → outline_to_msdf() → MSDF 图像（在 msdfer.rs 中调用）
```

## 坐标系统说明

- **字体坐标系**：使用 ttf_parser 的标准坐标系统，原点在基线左端，Y 轴向上
- **em 坐标系**：将字体坐标除以 `units_per_em`，得到与字号无关的归一化坐标
- **像素坐标系**：通过 `rasterize_transform` 变换，Y 轴翻转（光栅器使用左下原点，图像使用左上原点）
