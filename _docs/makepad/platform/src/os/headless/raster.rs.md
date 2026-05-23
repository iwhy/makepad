# `headless/raster.rs` — 软件光栅化渲染管线

## 概述

本文件实现了 headless 模式的核心渲染管线。它将 JIT 编译的着色器函数指针与 Makepad 的绘制树结合，在 CPU 上完整执行 GPU 渲染管线的等价操作：**遍历绘制树 → 解析着色器 → 执行顶点着色器 → 准备 uniform/纹理 → 调用虚拟 GPU 光栅化 → PNG 输出**。

---

## JIT 函数指针类型

### `VertexFn` — 顶点着色器入口

```rust
type VertexFn = unsafe extern "C" fn(
    geom_ptr: *const f32,    // 几何数据（位置、法线等）
    geom_len: u32,           // 几何数据 f32 数
    inst_ptr: *const f32,    // 实例数据
    inst_len: u32,           // 实例数据 f32 数
    uniform_ptrs: *const *const f32, // uniform 数组的指针数组
    uniform_lens: *const u32,        // 每个 uniform 数组的长度
    uniform_count: u32,              // uniform 数组数量
    varying_out: *mut f32,           // varying 输出缓冲区
    varying_len: u32,                // varying 缓冲区 f32 数
    out_pos: *mut [f32; 4],          // 输出裁剪坐标 [x, y, z, w]
);
```

每个顶点着色器调用处理一个顶点，接收几何数据、实例数据和 uniform 数组，输出裁剪坐标和 varying。

### `FragmentFn` — 片段着色器入口

```rust
type FragmentFn = unsafe extern "C" fn(rcx_ptr: *mut f32, rcx_f32s: u32) -> u32;
```

接收宿主预填充的 `RenderCx` 缓冲区，返回 `1`（写入像素）或 `0`（discard）。片段输出（`frag_fb0`）直接从缓冲区读取。

---

## 辅助数据类型

| 类型 | 说明 |
|------|------|
| `RowChunk` | 行分块：`(start, end)` 像素行范围 |
| `TextureConversionSignature` | 纹理缓存签名：保证缓存一致性 |
| `CachedTextureConversion` | 纹理转换缓存：记录格式转换后的 RGBA f32 数据 |
| `RenderProfile` | 渲染性能分析数据 |
| `TextureConversionCache` | `HashMap<usize, CachedTextureConversion>` 类型别名 |

---

## 纹理格式转换 — `headless_texture_info()`

Makepad 支持多种纹理格式，而 JIT 着色器只理解 `f32` 格式的纹理数据。此函数将各种 GPU 格式转换为统一的 `[data_ptr, data_len, width, height]` 四元组：

| 输入格式 | 转换方式 |
|----------|----------|
| `VecRGBAf32` / `VecMipRGBAf32` | 直接返回指针，零拷贝 |
| `VecBGRAu8_32` / `VecMipBGRAu8_32` | BGRA 字节 → RGBA f32（除以 255）|
| `VecCubeBGRAu8_32` | 立方体贴图 BGRA 字节 → RGBA f32 |
| `VecRu8` | 单通道字节 → RGBA f32（四个通道相同）|
| `VecRf32` | 单通道 f32 → RGBA f32（四个通道相同）|

使用**签名缓存**避免重复转换：`TextureConversionSignature` 包含 `(kind, width, height, data_ptr, data_len)`，数据未更新时直接返回缓存结果。

---

## 公共 API — `headless_render_all_passes()`

```
[入口] headless_render_all_passes(time)
  ↓
compute_pass_repair_order → 确定脏通道绘制顺序
  ↓
对每个 Window 类型的 draw_pass:
  ┌─────────────────────────────────────────────┐
  │  创建 Framebuffer(width, height)             │
  │  framebuffer.clear(clear_color, 1.0)          │
  │  headless_draw_pass(...)                      │
  └─────────────────────────────────────────────┘
  ↓
返回 Vec<(window_id, Framebuffer)>
```

步骤如下：
1. 调用 `compute_pass_repair_order` 获取需要重绘的所有通道
2. 对每个 `CxDrawPassParent::Window(window_id)` 的通道，根据窗口设置 `inner_size × dpi_factor` 计算帧缓冲尺寸
3. 设置正交投影矩阵、DPI 因子和时间 uniform
4. 创建并清空帧缓冲
5. 调用 `headless_draw_pass` 实际渲染
6. 收集所有结果 FBO，返回给调用者

---

## 绘制通道处理 — `headless_draw_pass()`

将 `draw_pass` 映射到 `draw_list`，然后委托给 `headless_render_view`。

---

## 视图递归处理 — `headless_render_view()`

这是遍历绘制树的核心函数，递归处理嵌套的 `SubList`：

对于每个 `draw_item`:
1. 如果是 `SubList`，递归调用 `headless_render_view`
2. 如果是 `DrawCall`，执行完整的渲染流水线：

### 逐 DrawCall 渲染流水线

1. **查找着色器**：通过 `shader_id` → `os_shader_id` → 获取 JIT 模块句柄
2. **解析函数指针**：从模块中加载 `makepad_headless_vertex` 和 `makepad_headless_fragment`
3. **准备 RenderCx 模板**：
   - 分配 `rcx_template` 缓冲区（大小由 `rcx_size` 指定）
   - 调用 `makepad_headless_fill_rcx` 填充 uniform 和纹理数据
   - uniform 来源：`DrawCallUniforms`, `DrawPassUniforms`, `DrawListUniforms`, `ScopeUniforms`, `DynamicUniforms`
4. **纹理转换**：遍历着色器的 textures 映射，对每个纹理调用 `headless_texture_info` 转换为 f32 格式
5. **获取几何数据**：从 `self.geometries[geometry_id]` 获取顶点和索引
6. **实例数据**：从 draw_item 实例数据中获取每个实例的 f32 槽位
7. **执行顶点着色器**：
   - 对每个实例的每个顶点调用 `vertex_fn`
   - 输出 `shaded_positions`（裁剪坐标数组）和 `shaded_varyings`（varying 数组）
8. **并行/串行光栅化选择**：
   - 如果三角形数 >= `parallel_min_tris` 且渲染线程数 > 1，使用线程池分块并行
   - 否则串行执行
9. **调用 `rasterize_instances_rows`** 执行光栅化：
   - 将帧缓冲按行分块（`compute_row_chunks`）
   - 每个线程处理一行块
   - 调用 `virtual_gpu.rs` 的 `rasterize_triangle_rows`

---

## 并行光栅化 — `rasterize_instances_rows()`（1231:335-583）

这是实际的"每实例三角光栅化循环"：

对于每个实例的每个三角形：
1. 从索引缓冲区读取三个顶点索引
2. 从 `shaded_positions` 和 `shaded_varyings` 中提取三个顶点的位置和 varying
3. 构建**片段着色器闭包**：
   - **无导数路径**: 直接写入 varying 到 `rcx_buf`，调用 `fragment_fn`
   - **有导数路径**: 在 2×2 像素块中分别计算 dx、dy 和主像素的 varying，调用三次 `fragment_fn`（仅在主像素写入）
4. 调用 `rasterize_triangle_rows` 执行实际像素扫描转换

---

## 工具函数

| 函数 | 说明 |
|------|------|
| `set_u32(buf, offset, val)` | 在字节缓冲区的指定偏移写入 u32 值 |
| `configured_render_threads()` | 从环境变量 `MAKEPAD_HEADLESS_THREADS` 读取线程数（默认 max(4, cpu_cores)）|
| `configured_parallel_min_tris()` | 从 `MAKEPAD_HEADLESS_PARALLEL_MIN_TRIS` 读取并行最小三角数 |
| `compute_index_chunks()` | 将索引范围均分（带余数分配）|
| `compute_row_chunks()` | 将帧缓冲高度分块（每块至少 32 行）|
| `write_varyings()` | 使用 `copy_nonoverlapping` 将 varying 数据写入 rcx 缓冲区 |

---

## PNG 编码 — `encode_png_rgba()`

将 RGBA8 `Vec<u8>` 编码为 PNG 格式：
1. 验证输入大小为 `width × height × 4`
2. 使用 `makepad_zune_png::PngEncoder`，设置 8bit 深度、RGBA 色彩空间
3. 返回编码后的 PNG 字节（可用于直接写入文件或通过网络协议传输）
