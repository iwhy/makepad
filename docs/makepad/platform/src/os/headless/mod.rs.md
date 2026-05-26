# `headless/mod.rs` — Makepad 无头平台层入口

## 概述

本文件是 headless（无头）平台模块的根模块，声明了所有子模块并定义了无头模式下操作系统相关的数据结构。Headless 模式的核心目标是在没有 GPU 和原生窗口系统的环境下运行 Makepad 渲染管线，这对 Flutter 嵌入、CI 测试、离线渲染等场景至关重要。

---

## 模块声明

```rust
mod event_loop;    // 无头事件循环（单帧/循环/标准输入协议）
mod jit;           // JIT 着色器编译（rustc → cdylib → dlopen）
mod raster;        // 软件光栅化器（CPU 执行顶点+片段着色器）
mod shader;        // 无头着色器状态与编译入口
mod virtual_gpu;   // 虚拟 GPU — Framebuffer、三角形填充、像素管线
```

---

## 空类型存根

以下是 GPU 后端的替代空类型，因为在 headless 模式下，这些概念由软件层直接处理：

| 类型 | 说明 |
|------|------|
| `CxOsDrawList` | 绘制列表 — 软件渲染中直接在 `headless_render_view` 中遍历 |
| `CxOsDrawCall` | 绘制调用 — 数据来源为 Rust 侧的 `DrawCall` 结构化数据 |
| `CxOsPass` | 渲染通道 — 软件渲染使用 `DrawPassId` 遍历 |
| `CxOsGeometry` | 几何数据 — 直接读写 `Cx::geometries` 中的 `Geometry` |
| `CxOsTexture` | 纹理状态 — 在 `raster::headless_texture_info` 中转换并缓存 |
| `CxOsUniformBuffer` | 统一缓冲 — 通过函数指针数组传入 JIT 着色器 |

---

## `CxOsDrawShader` — 着色器 JIT 元数据（核心结构）

此结构存储每个已编译 JIT 着色器模块的元数据，是 headless 渲染管线的关键纽带：

| 字段 | 类型 | 说明 |
|------|------|------|
| `source_hash` | `u64` | 着色器源代码的哈希值，用于去重和缓存 |
| `dylib_path` | `Option<PathBuf>` | JIT 编译产物的动态库路径 |
| `load_error` | `Option<String>` | 模块加载时的错误信息 |
| `module` | `Option<HeadlessLoadedModule>` | 通过 `libloading`/`dlopen` 加载的模块句柄 |
| `shader_version` | `Option<u32>` | 着色器版本号（从 JIT 模块的导出函数读取） |
| `varying_total_slots` | `usize` | varying 缓冲区的总 f32 槽位数 |
| `flat_varying_slots` | `usize` | 非插值 varying 槽位数（dyn/rust 实例数据） |
| `uses_derivatives` | `bool` | 片段着色器是否使用 dFdx/dFdy |
| `rcx_size` | `usize` | RenderCx 结构体的总字节大小 |
| `rcx_vary_offset` | `usize` | varying 区域的字节偏移 |
| `rcx_quad_mode_offset` | `usize` | quad_mode 区域的字节偏移 |
| `rcx_frag_offset` | `usize` | 片段输出（frag_fb0）的字节偏移 |
| `rcx_discard_offset` | `usize` | discard 标志的字节偏移 |

`rcx_*` 字段在 JIT 模块加载后查询导出函数获取，确保宿主端和 JIT 端的内存布局一致。这种设计让宿主可以直接操作 JIT 模块的 `RenderCx` 内存而不需要序列化。

---

## `CxOs` — 无头操作系统状态

```rust
pub struct CxOs {
    pub(crate) stdin_timers: PollTimers,    // 标准输入协议定时器
    pub(crate) start_time: Option<Instant>,  // 应用启动时间戳
    pub(crate) shader_jit: HeadlessShaderJit, // JIT 编译器实例
    pub(crate) frame_dir: Option<PathBuf>,   // 帧输出目录
    pub(crate) no_draw: bool,                // 禁用绘制（仅编译）
    pub(crate) no_draw_initialized: bool,    // no_draw 首次初始化标志
    pub(crate) draw_cycles: Option<usize>,   // 限制绘制循环次数
    pub(crate) render_pool: Option<MessageThreadPool<()>>, // 多线程渲染池
    pub(crate) render_pool_threads: usize,   // 渲染线程数
}
```

---

## 媒体 API 存根

`CxMediaApi` 的实现完全为空操作（no-op），因为无头模式不需要这些输入设备：
- `midi_input` / `midi_output` — 返回空的 `OsMidiInput` / `OsMidiOutput`
- `audio_output_box` / `audio_input_box` / `video_input_box` — 忽略传入的闭包
- `use_*` 方法 — 不执行任何操作

---

## `share_texture_for_presentable_image`

三个平台的 stub 实现，均返回零值/None：
- macOS → `u32` (0)
- Windows → `u64` (0)
- Linux → `Option<LinuxOwnedImage>` (None)

这些方法在有原生窗口时需要将 Makepad 纹理共享为平台可呈现图像，但在无头模式下没有实际窗口，因此返回空值。
