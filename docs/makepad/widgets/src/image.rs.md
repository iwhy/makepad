# `image.rs` — 图像控件

## 作用
通用的图像显示控件，支持本地文件、HTTP 资源、内存数据加载，支持多种缩放模式（`ImageFit`），支持精灵图动画（sprite sheet animation），支持异步加载和旋转。

## 自定义 Shader

### `DrawImage`
- 扩展 `DrawQuad`，添加 `image_texture`（纹理）、`opacity`、`image_scale`、`image_pan`、`async_load`（加载中状态）、`rotation`
- `rotate_2d_from_center` 函数：计算以中心为原点的旋转 UV 坐标
- `get_color_scale_pan`：应用 scale + pan + rotation 后采样纹理
- `pixel` 函数：混合 async_load 状态（加载中显示灰 #333），应用 opacity 和 premultiplied alpha

## 关键结构

### `ImageAnimation` — 动画模式枚举
| 变体 | 说明 |
|------|------|
| `Stop` | 不播放 |
| `Once` | 播放一次（逐帧步进） |
| `Loop` | 循环播放 |
| `Bounce` | 往返播放（正向后反向） |
| `Frame(f64)` | 指定帧索引 |
| `Factor(f64)` | 指定进度因子 [0,1] |
| `OnceFps(f64)` / `LoopFps(f64)` / `BounceFps(f64)` | 指定帧率的变体 |

### `Image`
| 字段 | 类型 | 说明 |
|------|------|------|
| `draw_bg` | `DrawImage`（`#[redraw]`） | 绘制对象 |
| `src` | `Option<ScriptHandleRef>` | 资源句柄 |
| `fit` | `ImageFit` | 缩放模式 |
| `texture` | `Option<Texture>` | GPU 纹理 |
| `animation` | `ImageAnimation` | 动画配置 |
| `next_frame` | `NextFrame` | 帧定时器 |

## 方法详解

### `load_from_resource`
- 从 `ScriptHandleRef` 加载图片数据。如果资源尚未加载完成（HTTP 请求进行中），等待下次绘制时重试
- 已加载完成时调用 `lazy_create_image_cache` 初始化图片缓存，然后调用 `load_image_from_data_async` 启动异步解码

### `handle_event`（Widget）
- 处理 `NetworkResponses` 网络响应事件
- 处理 `AsyncImageLoad` action（图片解码完成），调用 `process_async_image_load` 处理结果
- 处理 `NextFrame` 帧动画：
  - 根据 `animation` 模式计算下一帧索引（帧率模式使用 delta 时间累加，逐帧模式每次 +1）
  - 计算 sprite sheet 中的 UV 偏移：`xpos = ((frame % horizontal_frames) * width) / texture_width`
  - 支持 Bounce 模式（超过 `num_frames` 后反向播放）

### `draw_walk`（Widget）
- 调用 `load_from_resource` 确保资源加载
- 委托给 `draw_walk_image`

### `draw_walk_image`
- 处理 `Size::Fit { max }` 情况（NaN 高度处理）
- 根据 `ImageFit` 模式计算宽高：
  - `Size`：使用图像原始尺寸
  - `Stretch`：拉伸填充
  - `Horizontal`：宽度固定，高度按比例
  - `Vertical`：高度固定，宽度按比例
  - `Smallest`：取宽高中按比例缩放后较小者
  - `Biggest`：取较大者
- 动画纹理处理：计算 sprite sheet 的 scale（原始纹理 / 动画帧尺寸）
- 渲染纹理（RenderTarget）时翻转 Y 轴

### `size_in_pixels` / `has_texture`
- 查询纹理的原始像素尺寸和存在性

### `load_image_file_by_path_async`
- 异步加载磁盘图片文件。加载中显示占位尺寸并启动 `async_load` 动画

### `load_image_http_by_url_async`
- 从 URL 异步加载图片

### `load_image_from_data_async`
- 从内存数据异步加载图片

### ImageRef 方法
- `set_texture`：直接设置纹理（需在绘制事件内）
- `set_uniform`：设置 shader uniform
- `size_in_pixels` / `has_texture` / `load_*`：操作代理
