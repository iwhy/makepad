# Android Camera2 NDK FFI 绑定

## 概述

`acamera_sys.rs` 是 Android Camera2 NDK API 的 FFI 绑定。它声明了摄像头管理、设备控制、捕获会话、图像读取器等的 C 类型和外部函数。

通过 `#[link(name = "mediandk")]` 和 `#[link(name = "camera2ndk")]` 链接系统库。

## 不透明类型

| 类型 | 说明 |
|------|------|
| `ACameraManager` | 摄像头管理器 |
| `ACameraMetadata` | 摄像头元数据（特性） |
| `ACameraDevice` | 摄像头设备 |
| `ACaptureRequest` | 捕获请求 |
| `ACaptureSessionOutput` | 捕获会话输出 |
| `ACaptureSessionOutputContainer` | 输出容器 |
| `AImageReader` | 图像读取器 |
| `ACameraOutputTarget` | 输出目标 |
| `ACameraCaptureSession` | 捕获会话 |
| `AImage` | NDK 图像缓冲区 |

## 具象类型

| 类型 | 说明 |
|------|------|
| `ACameraIdList` | 摄像头 ID 列表（含数量） |
| `ACameraMetadata_rational` | 有理数（分子/分母） |
| `ACameraMetadata_const_entry` | 元数据条目（tag + union 数据） |
| `ACameraDevice_StateCallbacks` | 设备状态回调（断开、错误） |
| `ACameraCaptureSession_stateCallbacks` | 会话状态回调（关闭、就绪、活跃） |
| `ACameraCaptureSession_captureCallbacks` | 捕获过程回调 |
| `ACameraCaptureFailure` | 捕获失败信息 |
| `AImageReader_ImageListener` | 图像可用监听器 |
| `ACaptureSessionOutput_create` 相关 | 输出创建函数 |

## 关键常量

### 元数据 Tag
- `ACAMERA_LENS_FACING = 524293` — 摄像头朝向
- `ACAMERA_SENSOR_ORIENTATION = 393217` — 传感器旋转角度
- `ACAMERA_SCALER_AVAILABLE_STREAM_CONFIGURATIONS = 851978` — 可用流配置
- `ACAMERA_CONTROL_AE_TARGET_FPS_RANGE = 65541` — AE 目标帧率
- `ACAMERA_JPEG_QUALITY = 458756` — JPEG 质量

### 摄像头朝向
- `ACAMERA_LENS_FACING_FRONT = 0`
- `ACAMERA_LENS_FACING_BACK = 1`
- `ACAMERA_LENS_FACING_EXTERNAL = 2`

### 图像格式
- `AIMAGE_FORMAT_YUV_420_888 = 35`
- `AIMAGE_FORMAT_JPEG = 256`
- `AIMAGE_FORMAT_RGBA_8888 = 1`
- `AIMAGE_FORMAT_PRIVATE = 0x22`

### 请求模板
- `TEMPLATE_PREVIEW = 1` 到 `TEMPLATE_MANUAL = 6`

## 外部函数

### `mediandk` 库
- **AImageReader**：`new`、`newWithUsage`、`setImageListener`、`getWindow`、`delete`、`acquireNextImage`、`acquireLatestImage`
- **AImage**：`getPlaneData`、`getTimestamp`、`getPlaneRowStride`、`getPlanePixelStride`、`getFormat`、`getHardwareBuffer`、`delete`

### `camera2ndk` 库
- **ACameraManager**：`create`、`delete`、`getCameraIdList`、`getCameraCharacteristics`、`deleteCameraIdList`、`openCamera`
- **ACameraMetadata**：`free`、`getAllTags`、`getConstEntry`
- **ACameraDevice**：`createCaptureRequest`、`close`
- **捕获会话**：`createCaptureSession`、`setRepeatingRequest`、`stopRepeating`、`close`
- **输出/目标管理**：`ACaptureSessionOutput_create`/`free`、`ACaptureSessionOutputContainer_create`/`free`/`add`、`ACameraOutputTarget_create`/`free`
- **请求管理**：`ACaptureRequest_addTarget`/`removeTarget`/`free`、`ACaptureRequest_setEntry_u8`

### `nativewindow` 库
- `ANativeWindow_acquire` / `ANativeWindow_release`

## 实现说明

- 此文件为原始 FFI 声明层，不包含 Rust 包装逻辑。Camera2 调用实现在 `android_camera.rs` 中。
- `ANativeWindow` 和 `AHardwareBuffer` 类型别名指向 `ndk_sys` 中的定义，确保类型一致性。
- 回调函数指针类型（`ACameraDevice_StateCallback` 等）使用 `Option<unsafe extern "C" fn>`，允许传 `None`。
