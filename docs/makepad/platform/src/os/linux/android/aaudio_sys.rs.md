# AAudio FFI 绑定

## 概述

`aaudio_sys.rs` 是 Android AAudio 音频 API 的原始 FFI 绑定。它声明了不透明类型、函数类型别名、外部 C 函数以及所有 AAudio 常量和错误码。

AAudio 是 Android 原生音频 API，提供低延迟音频输入/输出。

## 类型定义

| 类型 | 说明 |
|------|------|
| `AAudioStreamBuilder` | 音频流构建器（不透明） |
| `AAudioStream` | 音频流句柄（不透明） |

### 枚举类型别名（均为 i32）

| 类型别名 | 用途 |
|----------|------|
| `aaudio_result_t` | 函数返回码 |
| `aaudio_direction_t` | 方向（输入/输出） |
| `aaudio_sharing_mode_t` | 共享模式（独占/共享） |
| `aaudio_format_t` | 音频格式（PCM_FLOAT 等） |
| `aaudio_stream_state_t` | 流状态 |
| `aaudio_data_callback_result_t` | 数据回调结果 |
| `aaudio_performance_mode_t` | 性能模式 |

### 回调类型

- `AAudioStream_dataCallback`：数据回调（音频数据填充/消费）
- `AAudioStream_errorCallback`：错误回调

## 外部函数

通过 `#[link(name = "aaudio")]` 链接 `libaaudio.so`：

### 构建器 API
- `AAudio_createStreamBuilder` — 创建流构建器
- `AAudioStreamBuilder_setDeviceId` / `setDirection` / `setSharingMode` / `setSampleRate` / `setChannelCount` / `setFormat` / `setBufferCapacityInFrames` / `setPerformanceMode` — 配置参数
- `AAudioStreamBuilder_setDataCallback` / `setErrorCallback` — 设置回调
- `AAudioStreamBuilder_openStream` — 打开流
- `AAudioStreamBuilder_delete` — 销毁构建器

### 流控制 API
- `AAudioStream_requestStart` / `requestStop` / `close`
- `AAudioStream_waitForStateChange` — 等待状态转换

### 查询 API
- `AAudioStream_getChannelCount` / `getSampleRate` / `getFormat`

## 常量

### 性能模式
- `AAUDIO_PERFORMANCE_MODE_LOW_LATENCY = 12`

### 格式
- `AAUDIO_FORMAT_PCM_FLOAT = 2`

### 方向
- `AAUDIO_DIRECTION_OUTPUT = 0`
- `AAUDIO_DIRECTION_INPUT = 1`

### 共享模式
- `AAUDIO_SHARING_MODE_EXCLUSIVE = 0`
- `AAUDIO_SHARING_MODE_SHARED = 1`

### 设备类型（AAUDIO_TYPE_*）
定义了 30 种设备类型常量（从 `AAUDIO_TYPE_UNKNOWN = 0` 到 `AAUDIO_TYPE_BLE_BROADCAST = 30`），涵盖：
- 内置设备（听筒、扬声器、麦克风）
- 有线设备（耳机、头戴式耳机、模拟/数字线路）
- 蓝牙设备（SCO、A2DP、BLE 耳机/扬声器/广播）
- HDMI 设备（HDMI、ARC、EARC）
- USB 设备（设备、附件、耳机）
- 其他（扩展坞、FM、IP、总线、远程子混音、回波参考等）

### 错误码
- `AAUDIO_OK = 0`
- 错误码范围 `-900` 到 `-880`，包括：`ERROR_BASE`、`ERROR_DISCONNECTED`、`ERROR_ILLEGAL_ARGUMENT`、`ERROR_INTERNAL`、`ERROR_INVALID_STATE`、`ERROR_INVALID_HANDLE`、`ERROR_UNIMPLEMENTED`、`ERROR_UNAVAILABLE`、`ERROR_NO_FREE_HANDLES`、`ERROR_NO_MEMORY`、`ERROR_NULL`、`ERROR_TIMEOUT`、`ERROR_WOULD_BLOCK`、`ERROR_INVALID_FORMAT`、`ERROR_OUT_OF_RANGE`、`ERROR_NO_SERVICE`、`ERROR_INVALID_RATE`

### 其他
- `AAUDIO_CALLBACK_RESULT_CONTINUE = 0`

## 实现说明

- 所有类型均为不透明结构体（opaque struct），仅在 Rust 侧作为指针使用。
- 此文件只包含 FFI 声明，不含实现逻辑。实际调用封装在 `android_audio.rs` 中。
- `AAUDIO_TYPE_*` 常量（`u32`）用于将 Java 侧获取的设备类型描述映射为用户可读字符串。
