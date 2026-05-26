# `gpu_info.rs` — GPU 信息与性能分级

## 概述

`GpuInfo` 在运行时采集 GPU 信息并评估其性能等级，用于 Makepad 渲染引擎的自适应管线决策（如着色器复杂度、纹理分辨率、渲染路径选择）。

## 核心类型

### `enum GpuPerformance`
五点性能分级：
- **`Tier1`**：最低端 GPU（如 Oculus Quest 1 的 Adreno 540）。
- **`Tier2`**：中低端 GPU（如 Oculus Quest 2 的 Adreno 610）。
- **`Tier3`**：中端 GPU（如 Intel 集成显卡、普通 Android 设备）。
- **`Tier4`**：高端 GPU（NVIDIA GeForce、AMD Radeon/ATI、Apple Silicon）。
- **`Tier5`**：预留顶级分级（如 RTX 3090+），尚未实现精确检测逻辑。

### `struct GpuInfo`
- **`min_uniform_vectors: u32`**：GPU 支持的最小 uniform 向量数。低值意味着 uniform 资源紧张，可能需要简化着色器。
- **`performance: GpuPerformance`**：运行时评定的性能等级。
- **`vendor: String`**：GPU 厂商名（标准化为小写）。
- **`renderer: String`**：GPU 渲染器字符串（标准化为小写）。

`Default` 实现保守地默认使用 `Tier4`（假设为较新硬件），`min_uniform_vectors` 为 1024。

## 方法

### `init_from_info(min_uniform_vectors, vendor, renderer)`
从底层图形 API（如 OpenGL/WGPU/Vulkan）获取的原始信息初始化 `GpuInfo`。实现逻辑：
1. 将 `vendor` 和 `renderer` 转为小写，便于后续字符串匹配。
2. 设置 `min_uniform_vectors`。
3. 以 `Tier3` 作为基线性能等级。
4. 通过包含式字符串匹配进行分级：
   - Qualcomm + "540" → Quest 1 (Tier1)
   - Qualcomm + "610" → Quest 2 (Tier2)
   - 含 "intel" → Tier3
   - 含 "ati" → Tier4
   - 含 "nvidia" → Tier4

作者注释指出该分级逻辑"极其粗略"，需要更精确的检测手段（如显存大小、核心数、基准测试结果）。

### `is_low_on_uniform_vectors() -> bool`
当 `min_uniform_vectors < 512` 时返回 `true`，指示 uniform 资源紧张。渲染引擎可依据此减少 uniform 数组大小、禁用某些着色器特性或回退到简化版本。
