# Android 音频实现

## 概述

`android_audio.rs` 实现了基于 AAudio 的 Android 音频输入/输出功能。管理音频设备枚举、流创建、数据回调和设备生命周期。

## 核心类型

### `AndroidAudioStreamData`
音视频流共享数据：
- `device_id` — 音频设备 ID
- `is_in_error_state` — 原子错误标志
- `change_signal` — 通知 UI 线程的信号
- `audio_buffer` — 音频数据缓冲区
- `actual_channel_count` — 实际通道数（从流查询）
- `channel_count` — 请求的通道数

### `AndroidAudioInputStream`
输入流包装（`data` + `input_fn` 回调）。

### `AndroidAudioOutputStream`
输出流包装（`data` + `output_fn` 回调）。

### `AndroidAudioDeviceDesc`
设备描述（`AAudio ID` + Makepad `AudioDeviceDesc`）。

### `AndroidAudioAccess`
音频访问管理器：
- `audio_input_cb` / `audio_output_cb` — 回调数组（每个设备槽位一个）
- `audio_inputs` / `audio_outputs` — 活动流列表
- `device_descs` — 已知设备描述
- `failed_devices` — 曾打开失败设备的集合

### `AndroidAudioError(String)`
AAudio 错误包装。

## 关键方法

### `AndroidAudioAccess::new(change_signal)`
创建 `Arc<Mutex<Self>>` 实例，初始化空的设备列表。

### `get_updated_descs() -> Vec<AudioDeviceDesc>`
通过 JNI 调用 Java 侧 `getAudioDevices` 枚举音频设备，解析 `"aaudio_id$$type$$channels$$name"` 格式的字符串。
始终生成默认输入/输出（各 2 通道，`aaudio_id = 0`）作为回退。

### `use_audio_inputs(devices)` / `use_audio_outputs(devices)`
管理活动音频流：
- 移除出错或不再需要的流
- 为新增的设备创建 `AndroidAudioInput`/`AndroidAudioOutput`
- 失败时记录到 `failed_devices` 并发送信号

### `AndroidAudioOutput::new(desc, change_signal, output_fn)`
创建 AAudio 输出流：
- 使用 `AAUDIO_DIRECTION_OUTPUT`、`AAUDIO_FORMAT_PCM_FLOAT`、48kHz、256 帧缓冲区
- 注册数据回调和错误回调
- 打开流并调用 `AAudioStream_requestStart`

### `AndroidAudioInput::new(desc, change_signal, input_fn)`
创建 AAudio 输入流（配置同上，方向为 `AAUDIO_DIRECTION_INPUT`）。

### 数据回调

输入流：从 `audioData` 读取 interleaved float 数据，通过 `copy_from_interleaved` 转换为 Makepad AudioBuffer，调用 `input_fn`。

输出流：调用 `output_fn` 填充 AudioBuffer，通过 `copy_to_interleaved` 写入 `audioData`。

## 错误处理

`AndroidAudioError::from()` 将 AAudio 错误码映射为人类可读的错误名称：
- 非负值直接返回 `Ok(result)`
- 负值匹配 `AAUDIO_ERROR_*` 常量，返回 `Err(AndroidAudioError(format!("AAudio error {} - {}", prefix, err_str)))`

## `aaudio_error!` 宏

语法：`aaudio_error!(AAudio_createStreamBuilder(&mut builder))`
- 接受一个表达式（函数调用）
- 评估结果并通过 `AndroidAudioError::from` 包装

## 实现说明

- 使用 `setup_builder` 共享构建器配置逻辑（除方向外输入/输出相同）
- 通过 `Arc<AtomicBool>` 跟踪错误状态，错误回调中设置标志，UI 线程通过 `is_in_error_state.swap()` 检测
- `AndroidAudioDeviceDesc` 的 `aaudio_id = 0` 表示默认设备
- 默认使用 48kHz 采样率、PCM_FLOAT 格式、共享模式
