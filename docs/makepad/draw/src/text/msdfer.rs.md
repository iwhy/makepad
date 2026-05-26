# `msdfer.rs` — 多通道符号距离场（MSDF）生成器

## 文件定位

该文件是 Makepad 文本渲染子系统的核心 SDF 生成模块，实现了 **Chlumský 式多通道符号距离场（MSDF）** 算法。MSDF 的核心思想是将字形轮廓的边缘按角度分配到 RGB 三个通道，避免单通道 SDF 在锐利拐角处的信号退化，从而在极低分辨率下仍能保持清晰的文字渲染。

## 关键常量

| 常量 | 值 | 含义 |
|------|-----|------|
| `CHANNEL_R/G/B` | 0b001/010/100 | 三个颜色通道的位掩码 |
| `COLOR_YELLOW/MAGENTA/CYAN` | RGB 组合 | 三条"子通道"的颜色标签（缺少一个主色） |
| `EDGE_COLORS` | `[CYAN, MAGENTA, YELLOW]` | 三种边缘颜色，用于边缘着色 |
| `QUADRATIC_FLATTEN_STEPS` | 64 | 二次贝塞尔曲线细分的步数 |
| `CUBIC_FLATTEN_STEPS` | 96 | 三次贝塞尔曲线细分的步数 |

## 核心结构

### `Msdfer`
- **`settings: Settings`** — 配置参数（内边距、半径、截止值、拐角角度阈值）
- **`outline_to_msdf()`** — 主入口，将 `GlyphOutline` 渲染为四通道 MSDF 图像

### `Settings`
- `padding: usize` — 输出图像四周的额外像素，用于 SDF 衰减区域
- `radius: f32` — SDF 的最大有效半径（像素单位）
- `cutoff: f32` — 截止值，与半径配合将距离映射到 [0,1]
- `corner_angle_threshold: f32` — 拐角检测的角度阈值（弧度）

## 核心算法流程（`outline_to_msdf`）

### 步骤 1：轮廓解析
`Shape::from_outline()` 将 `GlyphOutline` 中的命令序列转换为 `Shape` 结构，其中包含若干 `Contour`，每个 `Contour` 由 `Edge` 组成。`Edge` 有三种变体：
- `Linear` — 直线段
- `Quadratic` — 二次贝塞尔曲线
- `Cubic` — 三次贝塞尔曲线

遇到 `MoveTo` 或 `Close` 时，若当前边列表非空且首尾不重合，则自动补一条闭合直线段，然后推入 `Contour`。

### 步骤 2：边缘着色（Edge Coloring）
`color_shape_edges()` 执行 MSDF 的核心创新——为每条边分配一个 RGB 子通道：

1. **`detect_corners()`**：遍历每个 `Contour` 的相邻边，计算前一条边的末端切线与后一条边的始端切线。使用叉积与点积判断是否构成拐角。若切线向量为零向量（退化边），直接标记为拐角。

2. **`colors_from_corners()`**：从三种颜色（CYAN/MAGENTA/YELLOW）中依次分配。经过拐角时切换颜色，使连续的非拐角段颜色相同，拐角两侧颜色不同。

3. **`is_valid_coloring()`**：验证着色方案是否满足 MSDF 约束——拐角处两侧边的颜色必须不同。

4. **回退策略**：若三次起始颜色尝试均无法生成有效着色，则采用循环分配方案——每条边按索引取 `EDGE_COLORS[index % 3]`，若首尾颜色相同则调整最后一条边的颜色。

若整个轮廓无边构成拐角，则所有边着 `COLOR_WHITE`（RGB 全通道）。

### 步骤 3：展平（Flatten）
`flatten_shape()` 将所有曲线段离散化为 `FlatSegment`（起点、终点、颜色）：
- 线性边直接转换为一条 `FlatSegment`
- 二次贝塞尔曲线按 64 步细分
- 三次贝塞尔曲线按 96 步细分

所有点先通过 `rasterize_transform` 映射到图像像素坐标（原点偏移 + Y 轴翻转）。

### 步骤 4：逐像素扫描
对于输出图像的每个像素：

1. **计算有符号距离**：遍历所有 `FlatSegment`，对每条线段调用 `signed_distance()`：
   - 计算点到线段的投影参数 `param = (AQ·AB) / |AB|²`
   - 若投影在 (0,1) 内，正交距离为 `|AQ × AB_n|`，符号由叉积确定
   - 若投影在端点外，取端点到点的欧氏距离，符号由 `non_zero_sign(AQ × AB)` 决定
   - 返回 `SignedDistance{distance, dot}`，其中 `dot` 记录端点对齐度

2. **通道选择（Channel Selector）**：为 RGB 三通道各维护一个 `ChannelSelector`，记录该通道可见边集中的最近距离。`distance_is_better()` 的判定优先级：绝对值小者优先→dot 值小者优先。

3. **伪距离修正**（`distance_to_pseudo_distance`）：当最近点位于线段延长线上时，用正交伪距离替换欧氏距离，避免 MSDF 在长线段延长线方向产生伪影。

4. **回退逻辑**：若某通道未能找到有效距离，则使用全局最小距离（`min_distance`）作为该通道值。

5. **编码**：`encode_sdf_distance()` 将浮点距离映射到 `Unorm8`：
   ```
   value = 1.0 - (distance / radius + cutoff)
   ```
   然后调用 `sdfer::Unorm8::encode()` 将归一化值编码为 u8。

6. **输出**：以 BGRA 格式写入 `SubimageMut<Bgra>`，其中 B/G/R 分别对应 MSDF 的三个通道，A 通道存放标准全局 SDF。

## 辅助函数

### 几何计算
- `quadratic_point(p0, p1, p2, t)` — 二次贝塞尔曲线求值，使用 de Casteljau 公式展开
- `cubic_point(p0, p1, p2, p3, t)` — 三次贝塞尔曲线求值
- `is_point_almost_equal(a, b)` — 容差为 0.0001 的浮点相等判断

### 切线计算（`Edge::tangent_start` / `tangent_end`）
- 对直线段，直接返回终点减起点
- 对二次贝塞尔，优先取 p0→p1（起点切线）或 p1→p2（终点切线），若退化则取 p0→p2
- 对三次贝塞尔，依次尝试 p0→p1 → p0→p2 → p0→p3 作为降级方案

### 符号处理
- `non_zero_sign(value)` — 将浮点数的符号映射为 +1 或 -1，零值视为正
- `signed_pseudo_distance()` — 叉积绝对值除以线段长度，带符号

### `Vec2` 辅助结构
本地定义的二维向量类型，支持：
- `from_points()` — 从两点构造向量
- `normalized()` — 归一化（零向量返回零）
- `orthonormal(polarity)` — 返回垂直单位向量（控制方向）
- `dot()` / `cross()` — 点积与叉积
- `is_zero()` — 容差判零
- `length()` — 欧氏长度

## 与 `sdfer` 的关系

`encode_sdf_distance()` 调用 `sdfer::Unorm8::encode()`，利用了单通道 SDF 模块的数值编码能力，体现了代码复用。
