# `sdfer.rs` — 单通道符号距离场（SDF）生成器

## 文件定位

该文件实现了 Makepad 文本渲染的**单通道 SDF 生成**功能。与 `msdfer.rs` 的多通道方案不同，`Sdfer` 从字形覆盖图像（coverage bitmap）出发，利用外部 `sdfer` crate 的 ESDT（Exact Signed Distance Transform）算法生成标准的单通道 SDF。该方法适用于不需要超高清渲染效果的场景。

## `Sdfer` 结构

```rust
pub struct Sdfer {
    settings: Settings,
    reusable_buffers: Option<sdfer::esdt::ReusableBuffers>,
}
```

- **`settings`** — 配置参数（内边距、半径、截止值）
- **`reusable_buffers`** — 可复用的 ESDT 内部缓冲区，避免多次调用时重复分配内存

### `Settings`
- `padding: usize` — 输出图像四周的额外 SDF 衰减区域像素数
- `radius: f32` — SDF 有效半径（像素单位），控制模糊/锐利程度
- `cutoff: f32` — 归一化截止值，与半径共同决定"过渡带"宽度

## 核心方法

### `new(settings) -> Self`
用给定配置创建 `Sdfer` 实例，`reusable_buffers` 初始为 `None`，在首次调用后自动分配。

### `settings() -> Settings`
返回当前配置参数副本。

### `coverage_to_sdf(coverage, output)`
主入口方法，将二值/灰度覆盖图像转换为符号距离场：

1. **尺寸校验**：断言输出图像尺寸等于覆盖率图像尺寸加上 `2 * padding` 像素的边框。即 `output.size() = coverage.size() + 2 * padding`。

2. **像素数据复制**：逐像素从 `Subimage<'_, R>` 读取覆盖值（单通道 u8），转换为 `sdfer::Unorm8` 格式，填充到 `Vec` 中。注意这里的 `Unorm8` 是 `sdfer` crate 的自定义类型，不是 Rust 标准类型。

3. **ESDT 计算**：调用 `sdfer::esdt::glyph_to_sdf()`，传入：
   - `&mut coverage`：包装为 `sdfer::Image2d` 的灰度图像
   - `Params`：含 `pad`、`radius`、`cutoff` 等参数
   - `self.reusable_buffers.take()`：将上次使用的内部缓冲区归还给 `esdt` 复用

4. **缓冲区缓存**：将 ESDT 返回的 `ReusableBuffers` 存回 `self.reusable_buffers`，供下次调用复用，避免堆分配开销。

5. **结果回写**：遍历 ESDT 输出的 SDF 图像，将每个像素的 `Unorm8` 编码值写回 `output` 的对应位置（使用 `R::new()` 包装为单通道像素）。

## 数据流

```
覆盖图像 (R subimage) 
    → Vec<Unorm8> 
    → Image2d 
    → esdt::glyph_to_sdf() 
    → SDF Image2d 
    → SubimageMut<R> (输出)
```

## 与 `msdfer.rs` 的对比

| 特性 | `sdfer.rs` | `msdfer.rs` |
|------|-----------|-------------|
| 输入 | 二值/灰度覆盖图像 | 矢量轮廓（GlyphOutline） |
| 算法 | ESDT（精确 SDF 变换） | 暴力逐像素距离计算 + 边缘着色 |
| 通道数 | 单通道 | 三通道（RGB）+ 全局 SDF（A） |
| 抗锯齿 | 标准 SDF（拐角处信号衰减） | MSDF（拐角处保持清晰） |
| 性能 | 较快（算法复杂度 O(n)） | 较慢（逐像素 O(像素×线段)） |
| 内存 | 可复用缓冲区 | 每次重新展平所有线段 |

## 依赖关系

- 外部 `sdfer` crate 提供核心 ESDT 算法实现
- 内部使用 `geom::{Point, Size}` 进行像素坐标计算
- 内部使用 `image::{Subimage, SubimageMut, R}` 作为输入/输出图像接口
