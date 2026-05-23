# `camera_video_encoder.rs` — 视频编码器外壳

## 概述
该文件提供 `VideoEncoder` 结构体，作为 `MediaVideoEncoder` trait 的 Rust 侧封装。它统一处理编码器的创建、帧输入、关键帧请求和资源释放，同时暴露与平台无关的 API。

---

## `VideoEncoder`

### 字段
- **`source: VideoEncodeSource`**：记录编码输入源类型（摄像头/纹理/CPU帧），供调用方按需判断。
- **`codec: VideoCodec`**：记录当前编码器使用的编解码标准。
- **`backend: Box<dyn MediaVideoEncoder>`**：底层媒体插件编码器实现。

### `start(config, output) -> Option<Self>`

编码器创建工厂方法：
1. 验证配置有效性：宽高和帧率均需大于 0，无效则输出错误日志并返回 `None`。
2. 调用 `media_plugin()` 获取全局媒体插件实例，若不存在则返回 `None`。
3. 调用 `plugin.create_video_encoder(config, output)` 创建后端编码器，将 `output` 闭包传入以接收编码输出包。创建失败返回 `None`。
4. 返回包装后的 `VideoEncoder`。

### `push_frame(frame)`
将一帧摄像头帧数据送入编码器。`frame` 为 `CameraFrameRef`，由编码器后端处理格式转换和编码。

### `push_apple_pixel_buffer(pixel_buffer, timestamp_ns)`
macOS/iOS 平台特有方法，直接将 Apple 的 `CVPixelBufferRef` 送入编码器，避免额外的内存拷贝。仅在非 headless 且目标平台为 macOS/iOS 时编译。

### `request_keyframe() -> Result<(), VideoEncodeError>`
请求编码器输出一个关键帧（IDR 帧）。委派给后端实现。

### `stop()`
停止编码器。调用后端 `stop()` 方法，通常在 `Drop` 时自动调用。

### `Drop` 实现
析构时自动调用 `self.stop()` 确保编码器资源释放。

---

## 平台回调函数

### `on_android_h264_packet(encoder_id, pts_us, flags, data)`
Android 平台回调，当 MediaCodec H.264 编码器产出编码数据包时调用。将数据转发到媒体插件实例。

### `on_android_h264_error(encoder_id, message)`
Android 平台回调，当 MediaCodec H.264 编码器发生错误时调用。将错误信息转发到媒体插件实例处理。
