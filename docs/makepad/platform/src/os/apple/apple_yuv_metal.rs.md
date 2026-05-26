# apple_yuv_metal.rs — Metal YUV→RGB 着色器

**文件路径:** `platform/src/os/apple/apple_yuv_metal.rs`

**核心目的:** 在 Metal 上实现 YUV 到 RGB 的颜色空间转换着色器。主要用于视频播放，将解码后的 YUV 视频帧转换为 Metal 纹理用于渲染。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `DrawYuvMetal` | 实现 `DrawShaderTrait` 的 Metal YUV 绘制着色器 |
| `YuvMetalQuad` | YUV 绘制的四边形实例数据 |
| `YuvMetalHead` | 着色器头部常量数据 |

**关键方法:**
- `DrawYuvMetal::new(cx)` — 创建新的 YUV 着色器实例
- `DrawYuvMetal::draw_quad(...)` — 使用 YUV 纹理绘制四边形：
  - 绑定 Y、U、V 三个平面纹理
  - 设置颜色转换矩阵参数
  - 执行绘制调用

**着色器实现:**
- Metal 片段着色器将 YCbCr (BT.601/BT.709) 转换为 sRGB：
  ```
  R = Y + 1.402 * (V - 128)
  G = Y - 0.344 * (U - 128) - 0.714 * (V - 128)
  B = Y + 1.772 * (U - 128)
  ```
- 支持 BT.601 和 BT.709 色彩空间
- 支持有限的视频范围（16-235）和全范围（0-255）
- 使用 3 个独立的 Metal 纹理绑定（Y、U、V 平面）

**实现细节:**
- 使用 Apple 的 `MTLPixelFormatR8Unorm` 用于 8 位 Y/ U/ V 平面
- 颜色矩阵通过 uniform buffer 传递到着色器
- 支持 `kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange` 和 `FullRange`
- 与 `CVMetalTextureCache` 集成以高效处理来自视频解码器的像素缓冲区

**平台集成:** macOS 和 iOS/tvOS，使用 Metal 框架
