# Android 摄像头播放器

## 概述

`android_camera_player.rs` 将 Android NDK 摄像头作为视频播放源，对接 Makepad 标准的视频纹理系统。支持三种纹理更新模式：GL YUV 上传、CPU YUV 平面纹理、HardwareBuffer 外部纹理。

## 纹理模式

### `AndroidCameraTextureMode`
| 模式 | 说明 |
|------|------|
| `GlYuv` | GL 端 YUV → RGB 转换，通过 `upload_i420_slices_to_gl` |
| `CpuYuv` | CPU 端 YUV 数据直接填充 R8 纹理 |
| `HardwareBufferExternal` | 使用 AHardwareBuffer 零拷贝渲染 |

## 核心类型

### `AndroidCameraPlayer`
摄像头播放器状态管理：
- `video_id` / `texture_id` / `tex_y/u/v_id` — 视频和纹理标识符
- `input_id` / `format_id` — 摄像头输入和格式
- `width` / `height` — 视频尺寸
- `prepared` / `prepare_notified` — 就绪状态
- `native_preview` — 使用 Java SurfaceView 原生预览
- `texture_mode` — 纹理更新模式
- `yuv_rotation_steps` — YUV 旋转步数（基于传感器方向）
- `i420_frames` — 帧环形缓冲区（`CameraFrameLatest`）
- `hardware_buffer_frame` — 最新 AHardwareBuffer 帧
- `camera_access` — 摄像头访问引用

## 关键方法

### `new()`
构造播放器，注册帧回调到 `AndroidCameraAccess`：
- 根据 `native_preview`、`use_hardware_buffer_texture`、`use_cpu_plane_textures` 选择纹理模式
- 查询摄像头传感器方向，计算 YUV 旋转
- 注册 i420 帧回调或 hardware buffer 回调

### `check_prepared() -> Option<Result<PlaybackPrepared, String>>`
检查播放器是否已就绪：
- 原生预览模式：立即就绪
- HardwareBuffer 模式：等待第一帧
- CPU YUV 模式：等待第一帧数据

### `poll_frame(gl, textures) -> bool`
轮询新帧并上传到纹理：
- GlYuv 模式：使用 GL 上传 YUV→RGB
- CpuYuv 模式：直接填充 R8 平面纹理
- HardwareBuffer 模式：返回 false（由外部处理）

### `fallback_to_cpu_yuv()`
从 HardwareBuffer 模式回退到 CPU YUV 模式。当 HardwareBuffer 纹理不可用时调用。

### `take_hardware_buffer_frame()`
获取最新的硬件缓冲区帧。

### `set_preview_window()` / `cleanup()`
管理预览 Surface 和资源清理。

## 辅助函数

### `replace_r8_plane_texture(textures, texture_id, width, height, data)`
替换单平面 R8 纹理数据。处理 `VecRu8` 和 `VideoYuvPlane` 两种纹理格式。

## 实现说明

- `Drop` 实现中自动调用 `cleanup()`，确保摄像头资源被释放。
- `CameraFrameLatest` 使用 4 帧环形缓冲区缓存最新帧。
- `yuv_rotation_steps` 基于传感器方向（0°/90°/180°/270°）计算。
- `logged_first_hardware_buffer_consume` 和 `warned_waiting_for_first_frame` 用于调试日志。
