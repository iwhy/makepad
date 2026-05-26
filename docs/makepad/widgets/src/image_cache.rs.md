# `image_cache.rs` — 图像缩放模式和缓存引用

## 作用
定义 `ImageFit` 枚举（图像在容器中的缩放方式）和重导出底层 `makepad_draw` 中的图片缓存相关类型。

## 方法详解

### `ImageFit`
| 变体 | 说明 |
|------|------|
| `Stretch` | 拉伸填满容器 |
| `Horizontal` | 宽度匹配容器，高度按比例 |
| `Vertical` | 高度匹配容器，宽度按比例 |
| `Smallest` | 取水平/垂直缩放中较小者（完全可见） |
| `Biggest` | 取较大者（填满容器，可能被裁剪） |
| `Size` | 使用图像原始尺寸 |

### 重导出类型
- `handle_image_cache_network_responses`、`load_image_file_by_path_async`、`load_image_from_cache`、`load_image_from_data_async`、`load_image_http_by_url_async`、`process_async_image_load`、`AsyncImageLoad`、`AsyncLoadResult`、`ImageBuffer`、`ImageCache`、`ImageCacheImpl`、`ImageError`、`JpgDecodeErrors`、`PngDecodeErrors`
- 这些类型来自 `makepad_draw` 层，`image_cache.rs` 作为统一的前端入口重新暴露
