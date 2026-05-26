# `headless/virtual_gpu.rs` — 虚拟 GPU（软件光栅化核心）

## 概述

本文件实现纯 CPU 的虚拟 GPU，是 headless 渲染管线的核心光栅化引擎。它定义了一个完整的软件渲染管道：**裁剪坐标 → NDC → 屏幕空间 → 逐像素光栅化 → 片段着色 → 深度测试 → Alpha 混合**。所有计算在 CPU 上完成，无需任何 GPU 硬件。

---

## `Framebuffer` — 帧缓冲结构体

```rust
pub struct Framebuffer {
    pub width: usize,
    pub height: usize,
    pub color: Vec<[f32; 4]>, // RGBA 线性预乘 Alpha
    pub depth: Vec<f32>,      // 深度缓冲（范围 0~1）
}
```

### `Framebuffer::new(width, height)`
分配帧缓冲区：`color` 初始化为 `[0,0,0,0]`（全透明黑色），`depth` 初始化为 `1.0`（最远深度）。

### `Framebuffer::clear(color, depth)`
用指定的颜色和深度值填充整个帧缓冲。使用 `Vec::fill` 高效重置。

### `Framebuffer::to_rgba8()` — **Flutter 集成的关键方法**

将内部 `Vec<[f32;4]>` 线性 RGBA 格式转换为 `Vec<u8>` RSBA8 格式。执行以下步骤：

1. **反预乘 Alpha**: Makepad 内部使用预乘 Alpha（premultiplied alpha），即颜色的 RGB 分量已经乘以 A。输出时需要反预乘：
   ```rust
   let inv_a = if a > 0.0 { 1.0 / a } else { 0.0 };
   let r = (c[0] * inv_a).clamp(0.0, 1.0);
   ```
   透明像素（a=0）的 RGB 设为 0，避免除零。

2. **量化到 U8**: 将 `[0,1]` 范围的浮点数乘以 255 并四舍五入：
   ```rust
   out[base] = (r * 255.0).round() as u8;
   ```

这个方法生成的 `Vec<u8>` 可以直接编码为 PNG 或传递给 Flutter 的 `Image.fromBytes()`。这是无头渲染结果传递给外界的标准接口。

---

## `TriangleDerivatives` — 片段导数结构

```rust
pub struct TriangleDerivatives {
    pub dvary_dx: Vec<f32>,  // varying 对 x 的偏导数
    pub dvary_dy: Vec<f32>,  // varying 对 y 的偏导数
}
```

用于支持片段着色器中的 `dFdx(v)` / `dFdy(v)` 函数。通过在 2×2 像素块中计算相邻像素的 varying 差值来近似偏导数。

---

## `RasterScratch` — 光栅化暂存缓冲区

用于在帧的连续行之间复用 `Vec` 内存分配：

| 字段 | 说明 |
|------|------|
| `interp` | 插值后的 varying 值 |
| `interp_dx` | x+1 方向的 varying 值（用于导数计算）|
| `interp_dy` | y+1 方向的 varying 值（用于导数计算）|
| `derivs` | 最终导数结果 |

`ensure_vary_len()` 方法按需扩容暂存缓冲区，避免重复分配。

---

## `rasterize_triangle_rows()` — 核心三角形光栅化函数

这是整个虚拟 GPU 的核心函数，实现了完整的软件光栅化管线：

### 输入

- `width, height`: 帧缓冲尺寸
- `row_start, row_end`: 当前处理的像素行范围（支持分块并行）
- `color, depth_buf`: 帧缓冲切片
- `p0, p1, p2`: 三个顶点的 `[x, y, z, w]`（裁剪坐标）
- `vary0, vary1, vary2`: 三个顶点的 varying 数组
- `flat_slots`: 非插值槽位数（前 N 个 varying 直接从顶点 0 复制）
- `compute_derivatives`: 是否计算 dFdx/dFdy
- `scratch`: 暂存缓冲区
- `fragment_fn`: 片段着色器回调闭包

### 管线步骤

1. **NDC 转换**: 将裁剪坐标 `[x, y, z, w]` 转换为 NDC `[x/w, y/w, z/w]`，然后从 `[-1, 1]` 映射到屏幕空间 `[0, width]`：
   ```rust
   let sx = (ndc_x * 0.5 + 0.5) * w;
   let sy = (1.0 - (ndc_y * 0.5 + 0.5)) * h; // Y 轴翻转
   let sz = ndc_z; // 深度保持
   ```

2. **背面剔除/顶点排序**: 使用边缘函数 `edge()` 计算三角形面积。如果面积为负，交换 V1 和 V2 使其为正向（逆时针），确保一致的 Top-Left 规则。

3. **包围盒计算**: 计算三角形在屏幕空间的最小/最大包围盒，并裁剪到当前行范围。

4. **边缘函数增量**: 预计算每像素步进的边缘增量值 `e0_dx, e0_dy` 等，这些常量在整个三角形光栅化中不变。

5. **逐像素遍历**: 在包围盒内逐行逐列扫描：
   - 对每个像素计算三条边的 `edge()` 值
   - **Top-Left 规则**: 使用 GPU 标准的 Top-Left 填充规则，确保相邻三角形不产生缝隙或重叠
   - **深度测试**: 如果像素深度大于当前深度缓冲值，跳过（Less-or-Equal 模式）

6. **透视校正插值**: 使用 `interpolate_perspective()` 闭包执行透视校正 varying 插值：
   ```rust
   let a0 = w0 * inv_clip_w[0]; // w0 * 1/p0.w
   let denom = a0 + a1 + a2;
   out[i] = (a0*vary0[i] + a1*vary1[i] + a2*vary2[i]) / denom;
   ```
   这等价于 GPU 的透视校正插值 `interpolateAtSample`。

7. **Flat Varying 处理**: 索引 `< flat_slots` 的 varying 直接从 V0 复制，不做插值。这对应 GPU 中的 `flat` 插值限定符，用于逐实例（per-instance）数据。

8. **导数计算**（可选）:
   当 `compute_derivatives = true` 时，在 2×2 像素块内计算 dFdx/dFdy：
   - 评估相邻像素的 varying 值
   - `dvary_dx = interp(x+dx) - interp(x)`（dx 根据 lane_x 取 ±1）
   - `dvary_dy = interp(y+dy) - interp(y)`（dy 根据 lane_y 取 ±1）
   - Flat varying 的导数强制为零

9. **片段着色器调用**: 调用 `fragment_fn(varyings, derivatives, lane_x, lane_y, x, y)`，返回 `Option<[f32;4]>`（None 表示 discard）。

10. **预乘 Alpha 混合**: 使用 `blend_premul_src_over` 执行源覆盖混合：
    ```rust
    color[index] = [
        src.r + dst.r * (1 - src.a),
        src.g + dst.g * (1 - src.a),
        src.b + dst.b * (1 - src.a),
        src.a + dst.a * (1 - src.a),
    ];
    ```
    同时，仅当源 Alpha > 0.02 时更新深度缓冲，避免完全透明像素遮挡后续几何体。

---

## 辅助函数

| 函数 | 说明 |
|------|------|
| `edge(ax, ay, bx, by, px, py)` | 边缘函数，返回 `(px - ax)*(by - ay) - (py - ay)*(bx - ax)`。正值表示点在边右侧 |
| `is_top_left(ax, ay, bx, by)` | 判断边是否为 Top-Left 边。在 Y 向下增长的屏幕空间中，dy > 0 是上边，dy=0 且 dx < 0 是左边 |
| `edge_pass(e, top_left)` | 应用 Top-Left 规则的像素通过测试（e > -EPS 且 (e > 0 || top_left)）|
| `blend_premul_src_over(src, dst)` | 预乘 Alpha 的 SrcOver 混合，适用于 UI 渲染的透明叠加 |

---

## 与 headless 渲染管道的关系

```
shader.rs 生成 JIT Rust 源码
    ↓
jit.rs 编译为动态库并加载函数指针
    ↓
raster.rs 执行顶点着色器 → 准备几何数据
    ↓
virtual_gpu.rs: rasterize_triangle_rows()
    → NDC 转换 → 包围盒 → 逐像素边缘测试
    → 透视校正插值 → 深度测试 → 片段着色器
    → Alpha 混合 → 写入 Framebuffer
    ↓
raster.rs: to_rgba8() → encode_png_rgba() → PNG 输出
```
