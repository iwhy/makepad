# `texture.rs` — 纹理管理与格式定义

## 概述

`texture.rs` 定义了 Makepad 框架的纹理系统，包括 `Texture` 句柄类型、`TextureId` 标识符、`CxTexturePool` 纹理池、`TextureFormat`（约 18 种格式变体）、`CxTexture` 运行时纹理数据，以及纹理数据上传/更新/读取操作。视频纹理、渲染目标纹理、深度缓冲纹理、立方体贴图、Mipmap 纹理都在此文件定义。

---

## 核心类型

### `Texture`（第 9-10 行）

```rust
pub struct Texture(Rc<PoolId>);
```

引用计数句柄，通过 `Rc<PoolId>` 实现 Clone + Drop 安全。每次 Clone 增加引用计数，Drop 时减少——实际的池槽位由 `CxTexturePool` 管理，Texture 只是"票据"（通过内部 `PoolId` 的 `id` 和 `generation` 定位池槽）。

### `TextureId`（第 12-21 行）

```rust
pub struct TextureId(pub(crate) usize, u64);
```

- `usize`：池槽索引
- `u64`：槽位代数，用于检测悬空引用
- `Default`：返回 `TextureId(usize::MAX, 0)`，作为"无纹理"哨兵值（用于纯音频播放器）

### `CxTexturePool`（第 29-30 行）

```rust
pub struct CxTexturePool(pub(crate) IdPool<CxTexture>);
```

基于 `IdPool` 的纹理池。`IdPool` 支持带复用过滤的自动分配。

---

## 纹理格式枚举

### `TextureSize`（第 99-112 行）

```rust
pub enum TextureSize {
    Auto,
    Fixed { width: usize, height: usize },
}
```

渲染/深度纹理的大小策略：`Auto` 表示从渲染上下文获取，`Fixed` 指定固定尺寸。

### `TextureFormat`（第 114-208 行）

18 种格式变体：

#### CPU 上传的可向量纹理

| 变体 | 数据类型 | 用法 |
|------|----------|------|
| `VecBGRAu8_32 { width, height, data, updated }` | `Vec<u32>` | 标准 RGBA 图片（4x8bit），u32 表示一个像素 |
| `VecCubeBGRAu8_32 { width, height, data, updated }` | `Vec<u32>`（6 面） | 立方体贴图，面顺序：+X, -X, +Y, -Y, +Z, -Z |
| `VecMipBGRAu8_32 { width, height, data, max_level, updated }` | `Vec<u32>` + mip 层级 | 带 Mipmap 的 BGRA 纹理 |
| `VecMipRGBAf32 { width, height, data, max_level, updated }` | `Vec<f32>` + mip 层级 | 带 Mipmap 的 RGBA 浮点纹理 |
| `VecRGBAf32 { width, height, data, updated }` | `Vec<f32>` | RGBA 浮点纹理 |
| `VecRu8 { width, height, data, unpack_row_length, updated }` | `Vec<u8>` | 单通道 8bit 纹理 |
| `VecRGu8 { width, height, data, unpack_row_length, updated }` | `Vec<u8>` | 双通道 8bit 纹理 |
| `VecRf32 { width, height, data, updated }` | `Vec<f32>` | 单通道浮点纹理 |

#### 渲染目标纹理

| 变体 | 说明 |
|------|------|
| `DepthD32 { size, initial }` | D32 深度缓冲，`initial` 表示需要在首次渲染前初始化 |
| `RenderBGRAu8 { size, initial }` | BGRA 渲染目标 |
| `RenderCubeBGRAu8 { size, initial }` | 立方体贴图渲染目标 |
| `RenderRGBAf16 { size, initial }` | RGBA 半精度浮点渲染目标（HDR） |
| `RenderRGBAf32 { size, initial }` | RGBA 全精度浮点渲染目标 |

#### 共享纹理

| 变体 | 说明 |
|------|------|
| `SharedBGRAu8 { width, height, id, initial }` | 与 `shared_framebuf` 系统共享的 BGRA 纹理，使用 `PresentableImageId` 标识 |

#### 视频纹理

| 变体 | 说明 |
|------|------|
| `VideoYuvPlane` | YUV 平面纹理（I420 用 R8 三平面，NV12 用 R8 + RG8）。GPU 格式在上传时由后端设置 |
| `VideoExternal` | 不透明外部视频纹理（Android SurfaceTexture/OES，平台原生合成输出） |
| `VideoRgbaHardwareBuffer` | Android/Vulkan 导入的 RGBA `AHardwareBuffer` 相机纹理 |

---

## `TextureUpdated`——纹理更新追踪（第 375-406 行）

```rust
pub enum TextureUpdated {
    Empty,
    Partial(RectUsize),
    Full,
}
```

追踪纹理脏区域：
- `Empty`：无更新
- `Partial(RectUsize)`：部分区域变脏，RectUsize 定义脏矩形
- `Full`：整个纹理变脏

**方法：**
- `is_empty()`：判断是否无更新
- `update(dirty_rect)`：合并新的脏矩形。如果 `dirty_rect` 为 None，标记 Full；如果 `dirty_rect` 尺寸为零，保持 Empty；Partial 合并通过 `rect.union()` 扩展脏区域

---

## `TextureAlloc` / `TextureCategory` / `TexturePixel`（第 291-425 行）

# 内部辅助类型

- `TextureAlloc { category, pixel, width, height }`：GPU 分配的摘要信息
- `TextureCategory`：纹理类别枚举（Vec / VecMip / VecCube / Render / RenderCube / DepthBuffer / Shared / Video）
- `TexturePixel`：像素数据类型枚举（BGRAu8 / RGBAf16 / RGBAf32 / Ru8 / RGu8 / Rf32 / D32 / VideoYuvPlane / VideoExternal / VideoRgbaHardwareBuffer）

---

## `CxTexture`——运行时纹理数据（第 922-929 行）

```rust
pub struct CxTexture {
    pub format: TextureFormat,
    pub alloc: Option<TextureAlloc>,
    pub animation: Option<TextureAnimation>,
    pub os: CxOsTexture,
    pub previous_platform_resource: Option<CxOsTexture>,
}
```

- `format`：纹理格式定义（含宽高、数据、更新状态等）
- `alloc`：GPU 分配缓存（用于判断是否需要重新创建 GPU texture）
- `animation`：可选动画元数据
- `os`：平台层的 GPU 纹理资源
- `previous_platform_resource`：纹理槽被复用时保留的旧 OS 资源，用于延迟清理

### 方法

| 方法 | 说明 |
|------|------|
| `updated()` | 提取可向量纹理的更新状态。匹配所有 Vec* 变体，其他变体 panic |
| `initial()` | 提取渲染/深度/共享纹理的初始化标志 |
| `set_updated(updated)` | 设置可向量纹理的更新状态 |
| `set_initial(initial)` | 设置渲染/深度纹理的初始化标志 |
| `take_updated()` | 取出当前更新状态并重置为 Empty |
| `take_initial()` | 取出初始化状态并设回 false |
| `alloc_vec()` / `alloc_shared()` / `alloc_render(w, h)` / `alloc_depth(w, h)` / `alloc_video()` | 根据格式类型生成 TextureAlloc，如果与当前 alloc 不同则更新并返回 true |

---

## `CxTexturePool`——纹理池实现（第 32-70 行）

### `alloc(requested_format)`——分配纹理

**实现逻辑：**
1. 检查请求格式是否为视频纹理（通过 `is_video()`）
2. 创建 `CxTexture` 实例，设置 format
3. 调用 `IdPool::alloc_with_reuse_filter()`，复用过滤条件是：`is_video == item.format.is_video()`。这意味着视频纹理只能复用视频纹理槽，非视频纹理只能复用非视频纹理槽
4. 如果复用了一个旧槽，将旧槽的 `os` 资源存入新纹理的 `previous_platform_resource` 字段，保证延迟清理
5. 返回 `Texture(Rc::new(new_id))`

### Index/IndexMut（第 72-97 行）

通过 `TextureId` 访问纹理池：
- 从 `pool[index.0]` 获取槽位
- 验证 `generation == index.1`，不匹配时输出错误日志
- 返回/返回可变 `&CxTexture`

---

## `Texture` 公开方法

| 方法 | 说明 |
|------|------|
| `new(cx)` | 返回 `cx.null_texture()`（空纹理） |
| `new_with_format(cx, format)` | 使用指定格式分配新纹理。调用 `cx.textures.alloc(format)` |
| `texture_id()` | 返回 `TextureId(self.0.id, self.0.generation)` |
| `set_animation(cx, animation)` | 设置纹理动画元数据 |
| `animation(cx)` | 获取纹理动画元数据引用 |

### 数据操作方法（VecBGRAu8_32）

| 方法 | 说明 |
|------|------|
| `take_vec_u32(cx)` | 从纹理格式中取出 `Vec<u32>` 数据（消费式）。只适用于 `VecBGRAu8_32` |
| `swap_vec_u32(cx, vec)` | 交换纹理数据。无数据时先设为空 `Vec`。更新状态设为 `updated.update(None)` = Full |
| `set_data_u32(cx, w, h, new_data)` | 替换完整 BGRA texture 数据。同时更新 width、height，适用于动态分辨率变化 |
| `put_back_vec_u32(cx, new_data, dirty_rect)` | 归还/更新 BGRA texture 数据。更新状态合并脏矩形 |

### 数据操作方法（VecRu8 / VecRGu8）

| 方法 | 说明 |
|------|------|
| `take_vec_u8(cx)` | 取出 `Vec<u8>` 数据。适用于 `VecRu8` 和 `VecRGu8` |
| `put_back_vec_u8(cx, new_data, dirty_rect)` | 归还 u8 纹理数据。断言 data 在此之前必须被 take（防止丢失） |

### 数据操作方法（VecRf32 / VecMipRGBAf32 / VecRGBAf32）

| 方法 | 说明 |
|------|------|
| `take_vec_f32(cx)` | 取出 `Vec<f32>` 数据。适用于三种浮点格式 |
| `put_back_vec_f32(cx, new_data, dirty_rect)` | 归还浮点纹理数据。断言 data 在此之前必须被 take |

---

## `TextureFormat` 分类方法

| 方法 | 说明 |
|------|------|
| `is_shared()` | 是否为 `SharedBGRAu8` |
| `is_vec()` | 是否为任意可向量纹理变体（8 种） |
| `is_render()` | 是否为渲染目标（4 种） |
| `is_depth()` | 是否为深度缓冲 |
| `is_video()` | 是否为视频纹理（VideoYuvPlane / VideoExternal / VideoRgbaHardwareBuffer） |
| `is_video_external()` | 是否为外部视频纹理 |
| `is_video_rgba_hardware_buffer()` | 是否为 RGBA HardwareBuffer 视频纹理 |
| `vec_width_height()` | 获取可向量纹理的宽高（返回 `Option<(usize, usize)>`） |
| `is_compatible_with(other)` | 检查两个纹理格式是否兼容（目前只检查 video 标识是否相同） |

### `as_*_alloc()` 方法

| 方法 | 说明 |
|------|------|
| `as_vec_alloc()` | 将可向量纹理转换为 `TextureAlloc`。每种格式映射到对应的 `TextureCategory` 和 `TexturePixel` |
| `as_render_alloc(w, h)` | 将渲染目标格式转换为 `TextureAlloc`。使用 `TextureSize::width_height()` 确定最终尺寸 |
| `as_depth_alloc(w, h)` | 将深度格式转换为 `TextureAlloc` |
| `as_video_alloc()` | 将视频格式转换为 `TextureAlloc`（宽高设 0，GPU 加载时确定） |
| `as_shared_alloc()` | 将共享格式转换为 `TextureAlloc` |

---

## `TextureAnimation`（第 271-289 行）

```rust
pub struct TextureAnimation {
    pub width: usize,
    pub height: usize,
    pub num_frames: usize,
    pub frame_delays: Vec<f64>,
}
```

存储逐帧动画信息（帧延迟，单位 ms/帧）。

---

## Script 集成

`Texture` 实现了 `ScriptHook`、`ScriptApply`、`ScriptNew`：
- `ScriptNew::script_new(vm)` → 调用 `Texture::new(vm.cx_mut())`，即创建空纹理

---

## 测试

`texture_animation_tests`（第 279-289 行）
- 验证 `TextureAnimation::default()` 的 `frame_delays` 为空向量

---

## 总结

`texture.rs` 实现了完整的纹理生命周期管理。`Texture` 是引用计数的安全句柄，`CxTexturePool` 通过 `IdPool` 实现高效的槽位复用。纹理格式覆盖了从简单 BGRA 图片到 HDR 浮点纹理、深度缓冲、渲染目标、共享纹理和视频纹理的全场景。脏区域追踪（`TextureUpdated`）让部分更新成为可能，大幅减少 GPU 纹理上传带宽。
