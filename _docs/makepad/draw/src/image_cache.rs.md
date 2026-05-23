# `image_cache.rs` — 异步图像加载和缓存

**文件路径**: `draw/src/image_cache.rs` (1229 行)  
**核心作用**: 实现图像文件的异步解码、GPU 纹理上传和缓存管理。支持 PNG（含动画 PNG）、JPEG、WebP 和 GIF（含动画 GIF）格式的加载与解码。

---

## 一、核心数据结构

### `ImageBuffer`（第 21–27 行）

解码后的图像像素缓冲区，准备上传到 GPU 纹理：

```rust
pub struct ImageBuffer {
    pub width: usize,
    pub height: usize,
    pub data: Vec<u32>,                    // ARGB 格式像素（u32 打包）
    pub animation: Option<TextureAnimation>,// 动画信息（可选）
}
```

**像素格式**: `u32` 的 ARGB 格式排列——高 8 位 Alpha，接着 R、G、B 各 8 位。

**`animation`**: 当解码的是动画格式（GIF、APNG）时，包含帧数、帧延迟等信息。非动画图为 `None`。

### `ImageCacheEntry`（第 369–372 行）

缓存条目状态枚举：

```rust
pub enum ImageCacheEntry {
    Loaded(Texture),        // 已解码并上传到 GPU
    Loading(usize, usize),  // 正在加载（存储占位尺寸 w, h）
}
```

- `Loaded`: 纹理已就绪
- `Loading`: 后台线程正在解码，存储已知的宽高供占位布局使用

### `AsyncImageLoad`（第 374–378 行）

异步加载结果的消息载体，通过 `Cx::post_action()` 发送到主线程：

```rust
pub struct AsyncImageLoad {
    pub image_path: PathBuf,
    pub result: RefCell<Option<Result<ImageBuffer, ImageError>>>,
}
```

### `ImageCache`（第 380–399 行）

全局图像缓存管理器：

```rust
pub struct ImageCache {
    pub map: HashMap<PathBuf, ImageCacheEntry>,          // 路径→缓存的映射
    pub thread_pool: Option<TagThreadPool<PathBuf>>,      // 后台解码线程池
    pub pending_http_requests: HashMap<LiveId, PathBuf>,  // 进行中的 HTTP 请求
}
```

### `ImageError`（第 402–413 行）

图像解码错误枚举：

```rust
pub enum ImageError {
    EmptyData,
    InvalidPixelAlignment(usize),  // 不支持的像素对齐（每像素不是1/2/3/4字节）
    JpgDecode(JpgDecodeErrors),
    PathNotFound(PathBuf),
    PngDecode(PngDecodeErrors),
    GifDecode(GifDecodeErrors),
    WebpDecode(WebpDecodeErrors),
    UnsupportedFormat,
    Http(String),
}
```

### `AsyncLoadResult`（第 415–418 行）

```rust
pub enum AsyncLoadResult {
    Loading(usize, usize),  // 正在加载中，已知宽高
    Loaded,                 // 已加载完成
}
```

---

## 二、`ImageBuffer` 解码方法

### `ImageBuffer::new(in_data, width, height)`（第 30–85 行）

从原始字节数组构造 `ImageBuffer`，支持 4 种像素对齐：

| 每像素字节数 | 输入格式 | 输出格式 |
|-------------|---------|---------|
| 4（RGBA） | `[R, G, B, A, ...]` | `A<<24 \| R<<16 \| G<<8 \| B` |
| 3（RGB） | `[R, G, B, ...]` | `0xFF000000 \| R<<16 \| G<<8 \| B`（Alpha=255） |
| 2（RA） | `[R, A, ...]` | `A<<24 \| R<<16 \| R<<8 \| R`（灰度+Alpha） |
| 1（R） | `[R, ...]` | `0xFF000000 \| R<<16 \| R<<8 \| R`（灰度，Alpha=255） |

### `ImageBuffer::into_new_texture(self, cx)`（第 87–99 行）

将 `ImageBuffer` 上传为 GPU `Texture`：
1. 调用 `Texture::new_with_format(TextureFormat::VecBGRAu8_32)` 创建纹理
2. 设置动画信息 `self.animation`

### `ImageBuffer::from_png(data)`（第 101–124 行）

解码 PNG 图像（使用 `makepad_zune_png`）：
1. 创建 `PngDecoder`，调用 `decode_headers()`
2. 检查 `is_animated()` — 动画 PNG 走 `decode_animated_png()`
3. 非动画 PNG：调用 `decode()` 获取原始数据，获取尺寸，通过 `ImageBuffer::new()` 转换

### `ImageBuffer::decode_animated_png(decoder)`（第 126–222 行）

解码动画 PNG（APNG），将帧平铺到一张大纹理上：

1. 获取 `actl_info`（动画控制信息），包含帧数
2. 计算总纹理尺寸：`total_width = fits_horizontal * width`，其中 `fits_horizontal = max_texture_width / frame_width`
3. 循环解码每一帧：
   - `decode_headers()` + `decode_raw()`
   - `post_process_image()` 处理帧差异
   - 按 4/3 通道将帧写入 `final_buffer.data` 的正确位置
   - `cx += width`，超水平边界时 `cy += height; cx = 0`
4. 仅初始化 `animation` 结构体，**不设置 frame_delays**（PNG 的 fcTL 包含延迟信息，但当前代码未解析）

**注意**: 从测试 `test_from_png_animated_does_not_populate_frame_delays`（第 799 行）可以看出，动画 PNG 的 `frame_delays` 列表为空。这表明 APNG 帧延迟解析尚待完善。

### `ImageBuffer::from_webp(data)`（第 224–237 行）

解码 WebP 图像（使用 `makepad_webp`）：
1. 创建 `WebPDecoder`，获取尺寸和输出缓冲区大小
2. `read_image()` 解码到缓冲区
3. 通过 `ImageBuffer::new()` 转换

### `ImageBuffer::from_gif(data)`（第 239–349 行）

解码 GIF 图像（使用 `makepad_gif`），支持动画：

1. **单帧 GIF**（第 310–313 行）：直接通过 `ImageBuffer::new()` 返回，`animation = None`
2. **多帧动画 GIF**（第 315–348 行）：

**GIF 帧合成逻辑**（第 250–307 行）：
- 使用全尺寸 `canvas` 追踪当前帧的累积像素状态
- 每帧按 `(frame_left, frame_top)` 偏移写入
- 仅非透明像素（`rgba[3] != 0`）覆盖画布
- **帧后处理**：
  - `DisposalMethod::Background`：清空帧区域为透明
  - `DisposalMethod::Previous`：恢复到帧前快照
  - `DisposalMethod::Keep`（默认）：保留

**帧延迟处理**: 
- `frame.delay == 0` 时强制设为 `0.1` 秒（第 252–254 行），避免除以零
- 正常 `delay` 乘以 `0.01` 转换为秒

**帧平铺**（第 315–348 行）：
- 与 APNG 相同的平铺策略：`fits_horizontal = max_texture_width / width`
- 计算 `total_width` 和 `total_height`
- 逐帧写入 `final_buffer.data`

### `ImageBuffer::from_jpg(data)`（第 351–366 行）

解码 JPEG 图像（使用 `makepad_zune_jpeg`）：
1. 创建 `JpegDecoder`，调用 `decode()` 解码
2. 从 `decoder.info()` 获取尺寸
3. 通过 `ImageBuffer::new()` 转换

---

## 三、格式检测

### `detect_image_format(data)`（第 468–487 行）

通过魔数（magic bytes）检测图像格式：

| 格式 | 魔数 |
|------|------|
| PNG | `\x89PNG\r\n\x1a\n`（8 字节） |
| JPEG | `\xFF\xD8`（2 字节） |
| WebP | `RIFF....WEBP`（12 字节） |
| GIF | `GIF87a` 或 `GIF89a`（6 字节） |

### `detect_image_format_from_path_and_data(path, data)`（第 489–508 行）

**优先使用魔数检测**，魔数无法识别时回退到文件扩展名。

### `decode_image_buffer(path, data)`（第 510–520 行）

高层次的解码调度函数：检测格式 → 分发到对应的解码方法。

### `image_size_by_data(data, path)`（第 522–570 行）

仅解码头部获取图像尺寸（不解码完整像素），用于 `Loading(w, h)` 状态的占位。

---

## 四、异步加载系统

### `ensure_image_cache(cx)` / `ensure_image_cache_inner(cx)`（第 572–576 行，第 894–896 行）

确保 `ImageCache` 全局对象已存在。使用 `cx.has_global::<ImageCache>()` 检查，不存在时通过 `cx.set_global(ImageCache::new())` 创建。

### `ensure_thread_pool(cx)`（第 842–848 行）

确保线程池已创建。线程数策略：`max(1, cpu_cores - 2)`，在 `cpu_cores.max(3) - 2` 基础上取正（至少保留一个主线程和一个线程给线程池）。注意原始代码 `threads = cx.cpu_cores().max(3) - 2`，所以当 CPU 核心数为 2 时线程数为 1，为 1 时线程数可能为负（但 `TagThreadPool` 内部会处理）。

### `spawn_decode_job(cx, image_path, data)`（第 850–892 行）

**核心异步解码入口**：

1. 确保线程池已创建
2. 通过 `thread_pool.execute_rev(image_path, move |image_path| {...})` 提交解码任务
3. 在后台线程中：
   - 调用 `decode_image_buffer()` 解码
   - 打印调试日志（如果 `MAKEPAD_GLTF_TEX_DEBUG` 环境变量设置）
   - 通过 `Cx::post_action(AsyncImageLoad { ... })` 将结果发回主线程

`execute_rev` 是"可撤销的执行"——如果新任务排队的路径与待处理任务相同，旧任务会被取消。

### `load_image_from_data_async(cx, image_path, data)`（第 949–993 行）

异步加载内存中的图像数据：

1. **缓存检查**：查找 `ImageCache.map`，已加载则返回 `Loaded`，加载中则返回 `Loading(w, h)`
2. **wasm 同步解码**（第 964–976 行）：WebAssembly 平台不支持线程池，直接同步解码并上传
3. **headless 同步解码**（第 966–976 行）：`MAKEPAD=headless` 环境变量时也同步解码，确保单帧渲染可立即输出纹理
4. **异步解码**：先通过 `image_size_by_data()` 获取占位尺寸，插入 `Loading(w, h)` 到缓存，然后 `spawn_decode_job()`

### `load_image_file_by_path_async(cx, image_path)`（第 995–1008 行）

从文件系统异步加载图像：
1. 缓存检查（同 `load_image_from_data_async`）
2. `std::fs::read()` 读取文件
3. 委托给 `load_image_from_data_async()`

### `load_image_http_by_url_async(cx, url)`（第 1010–1031 行）

从 HTTP URL 异步加载图像：
1. 缓存检查
2. 生成唯一 `LiveId` 作为请求标识
3. 插入 `Loading(1, 1)`（占位，真实尺寸未知）
4. 向 `pending_http_requests` 注册映射
5. 调用 `cx.http_request(request_id, HttpRequest::new(url, GET))`

### `process_async_image_load(cx, image_path, result)`（第 898–939 行）

**主线程回调**，处理后台解码完成的结果：
1. 如果解码成功：调用 `data.into_new_texture(cx)` 上传到 GPU，将 `Loading` → `Loaded`
2. 如果解码失败：从缓存中移除该路径

### `handle_image_cache_network_responses(cx, event)`（第 1033–1089 行）

**HTTP 网络响应处理器**：

1. 遍历 `NetworkResponsesEvent` 中的所有响应
2. **`HttpError`**：从 `pending_http_requests` 移除，删除缓存条目
3. **`HttpResponse` / `HttpStreamComplete`**：
   - 检查状态码是否在 200–299 范围
   - 如果成功且有响应体，添加到 `decode_queue`
   - 失败则删除缓存条目
4. 遍历 `decode_queue`，为每个成功的 HTTP 响应调用 `load_image_from_data_async()` 启动解码

此函数处理所有网络事件类型，但只处理 HTTP 相关的（忽略 `WsMessage` 等 WebSocket 事件）。

---

## 五、`ImageCacheImpl` Trait（第 1091–1229 行）

提供一组有默认实现的方法，供需要图像加载能力的 Widget 实现：

| 方法 | 功能 |
|------|------|
| `get_texture(id)` / `set_texture(texture, id)` | 抽象纹理存取（由实现者提供存储） |
| `lazy_create_image_cache(cx)` | 延迟创建全局缓存 |
| `load_png_from_data(cx, data, id)` | 同步加载 PNG |
| `load_jpg_from_data(cx, data, id)` | 同步加载 JPEG |
| `process_async_image_load(cx, path, result)` | 处理异步加载结果 |
| `load_image_from_cache(cx, path, id)` | 从缓存加载纹理 |
| `load_image_from_data_async_impl(cx, path, data, id)` | 异步加载内存数据 |
| `load_image_file_by_path_async_impl(cx, path, id)` | 异步加载文件路径 |
| `load_image_http_by_url_async_impl(cx, url, id)` | 异步加载 HTTP URL |
| `load_image_file_by_path_and_data(cx, data, id, path)` | 同步加载文件并加入缓存 |
| `load_image_file_by_path(cx, path, id)` | 加载文件（优先缓存） |
| `load_image_dep_by_path(cx, path, id)` | 加载 Makepad 依赖资源 |

每个 "xxx_impl" 方法在异步加载完成后自动调用 `load_image_from_cache()` 将结果写入 `set_texture()`。

---

## 六、测试套件（第 578–839 行）

### 辅助函数

- `single_frame_gif()` / `animated_gif()` / `zero_delay_gif()` — 使用 `makepad_gif::Encoder` 构建 GIF 测试数据
- `animated_png()` — 手动构建 APNG 二进制数据（含 IHDR、acTL、fcTL、IDAT、fdAT、IEND 块）
- `zlib_stored()` / `crc32()` / `adler32()` — PNG 块构建工具
- `rgba_frame()` / `fctl()` — APNG 帧数据构建

### 测试用例

| 测试 | 验证点 |
|------|--------|
| `test_detect_image_format_recognises_gif89a` | GIF 魔数检测 |
| `test_detect_image_format_recognises_gif87a` | GIF 87a 格式检测 |
| `test_detect_image_format_still_recognises_png_after_gif_branch` | PNG 魔数优先级正确 |
| `test_detect_image_format_from_path_and_data_falls_back_to_gif_extension` | 扩展名回退 |
| `test_from_gif_decodes_single_frame` | 单帧 GIF 无动画 |
| `test_from_gif_packs_animated_frames_into_atlas` | 动画 GIF 帧平铺和延迟解析 |
| `test_from_gif_single_frame_has_no_frame_delays` | 单帧无延迟信息 |
| `test_from_gif_zero_delay_normalised_to_100ms` | 零延迟帧归一化 |
| `test_from_png_animated_does_not_populate_frame_delays` | APNG 延迟列表为空 |
| `test_from_gif_rejects_truncated_data` | 损坏数据的错误处理 |
| `test_decode_image_buffer_rejects_random_bytes_as_unsupported` | 随机数据的错误处理 |
| `test_makepad_gif_is_only_a_dependency_of_makepad_draw` | 验证 `makepad-gif` 的唯一依赖是 `makepad-draw` |

---

## 七、调试支持

### `image_decode_debug_enabled()`（第 434–441 行）

通过环境变量 `MAKEPAD_GLTF_TEX_DEBUG` 控制调试输出。值不为 `"0"` 时启用。

### `decode_timing_start()`（第 444–456 行）

返回 `Instant::now()` 用于计时，但在 wasm 平台返回 `None`。

### 调试日志示例

```
ImageCache: decode_start key=textures/foo.png bytes=12345
ImageCache: decode_done key=textures/foo.png elapsed_ms=12.3 ok 256x256
ImageCache: gpu_commit key=textures/foo.png elapsed_ms=0.5 size=256x256
```

---

## 八、流程图：完整异步加载流水线

```
load_image_file_by_path_async(path)
  │
  ├─ 缓存命中 → Loaded → 返回
  ├─ Loading 中 → Loading(w,h) → 返回
  └─ 未缓存
       │
       ├─ wasm/headless → 同步解码 → 上传 GPU → Loaded
       │
       └─ 异步路径
            ├─ image_size_by_data() → 获取占位宽高
            ├─ 插入 Loading(w,h) 到缓存
            ├─ spawn_decode_job()
            │    └─ 线程池 execute_rev()
            │         └─ decode_image_buffer()
            │              └─ Cx::post_action(AsyncImageLoad)
            │                   │
            └─ 返回 Loading(w,h) 给调用者
                            │
                   主线程事件循环
                            │
                   接收到 AsyncImageLoad
                            │
                   process_async_image_load()
                    ├─ 成功 → into_new_texture() → Loaded
                    └─ 失败 → 从缓存移除
```
