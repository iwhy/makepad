# pulse_audio.rs

One-liner (EN): PulseAudio audio backend — device enumeration, audio input (recording) and output (playback) stream management via the threaded mainloop API.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/pulse_audio.rs` (694 行)
- **核心作用**: 实现基于 PulseAudio 的音频输入/输出后端。使用 `pa_threaded_mainloop` 线程安全 API 管理音频流生命周期（创建、回调、销毁），并通过 `AlsaAudioAccess` 共享的回调数组与上层音频系统集成。

## 类型/结构体

### `PulseAudioDesc` — 设备描述封装

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `String` | PulseAudio 设备名称 |
| `desc` | `AudioDeviceDesc` | 音频设备描述 |

### `ContextState` — 上下文状态枚举

`Connecting` | `Ready` | `Failed`

### `PulseInputStream` — 音频输入流

| 字段 | 类型 | 说明 |
|------|------|------|
| `device_id` | `AudioDeviceId` | 设备标识 |
| `stream` | `*mut pa_stream` | PulseAudio 流指针 |

### `PulseInputStruct` — 输入流运行时状态

| 字段 | 类型 | 说明 |
|------|------|------|
| `device_id` | `AudioDeviceId` | 设备标识 |
| `input_fn` | `Arc<Mutex<Option<AudioInputFn>>>` | 输入回调 |
| `audio_buffer` | `AudioBuffer` | 音频缓冲区 |
| `ready_state` | `AtomicU32` | 就绪状态（1=就绪, 2=失败） |
| `main_loop` | `*mut pa_threaded_mainloop` | 主循环指针 |

### `PulseOutputStream` — 音频输出流

| 字段 | 类型 | 说明 |
|------|------|------|
| `device_id` | `AudioDeviceId` | 设备标识 |
| `stream` | `*mut pa_stream` | PulseAudio 流指针 |

### `PulseOutputStruct` — 输出流运行时状态

| 字段 | 类型 | 说明 |
|------|------|------|
| `device_id` | `AudioDeviceId` | 设备标识 |
| `output_fn` | `Arc<Mutex<Option<AudioOutputFn>>>` | 输出回调 |
| `write_byte_count` | `usize` | 可写入字节数 |
| `clear_on_read` | `bool` | 首次写入标记 |
| `ready_state` | `AtomicU32` | 就绪状态 |
| `audio_buffer` | `AudioBuffer` | 音频缓冲区 |
| `main_loop` | `*mut pa_threaded_mainloop` | 主循环指针 |

### `PulseAudioAccess` — PulseAudio 访问入口

| 字段 | 类型 | 说明 |
|------|------|------|
| `audio_input_cb` | `[Arc<Mutex<Option<AudioInputFn>>>; MAX_AUDIO_DEVICE_INDEX]` | 输入回调数组（从 AlsaAudioAccess 共享） |
| `audio_output_cb` | `[Arc<Mutex<Option<AudioOutputFn>>>; MAX_AUDIO_DEVICE_INDEX]` | 输出回调数组 |
| `buffer_frames` | `usize` | 缓冲区帧数 (256) |
| `audio_outputs / audio_inputs` | `Vec<PulseOutput/InputStream>` | 活跃流列表 |
| `device_query` | `Option<PulseDeviceQuery>` | 设备查询状态 |
| `device_descs` | `Vec<PulseAudioDesc>` | 缓存设备描述 |
| `change_signal` | `SignalToUI` | 设备变化信号 |
| `context_state` | `ContextState` | 上下文连接状态 |
| `main_loop` | `*mut pa_threaded_mainloop` | 线程化主循环 |
| `main_loop_api` | `*mut pa_mainloop_api` | 主循环 API 表 |
| `context` | `*mut pa_context` | PulseAudio 上下文 |
| `self_ptr` | `*const Mutex<PulseAudioAccess>` | 自引用指针 |
| `failed_devices` | `HashSet<AudioDeviceId>` | 失败设备集合 |

### `PulseDeviceDesc` / `PulseDeviceQuery` — 设备枚举辅助

`PulseDeviceQuery` 包含 `sink_list`、`source_list`、`default_sink`、`default_source` 和 `main_loop`。

## 关键方法

### PulseInputStream

| 方法 | 说明 |
|------|------|
| `new(device_id, name, index, pulse) -> Self` | 创建输入流：锁主循环→创建 pa_stream→设置状态/读取回调→connect_record→等待就绪→解锁 |
| `terminate(pulse)` | 终止输入流：锁主循环→清空读取回调→disconnect→unref→解锁 |
| `recording_stream_read_callback(stream, _nbytes, input_ptr)` | C 回调：`pa_stream_peek` 读取 interleaved f32 数据 → `AudioBuffer::copy_from_interleaved` → 调用 input_fn |
| `recording_stream_state_callback(stream, input_ptr)` | C 回调：跟踪流状态（READY/FAILED/TERMINATED），信号通知等待线程 |

### PulseOutputStream

| 方法 | 说明 |
|------|------|
| `new(device_id, name, index, pulse) -> Option<Self>` | 创建输出流：锁主循环→创建 pa_stream→设置状态回调→connect_playback (CORKED)→等待就绪→设置写入回调→cork(0) 启动→解锁 |
| `terminate(pulse)` / `terminate_stream(stream, pulse)` | 锁主循环→清空写入回调→disconnect→unref→解锁 |
| `playback_stream_write_callback(stream, _nbytes, output_ptr)` | C 回调：`pa_stream_begin_write` → 调用 output_fn 填充 AudioBuffer → `copy_to_interleaved` → `pa_stream_write` (含首次 SEEK_RELATIVE_ON_READ) |
| `playback_stream_state_callback(stream, output_ptr)` | C 回调：跟踪流 READY/FAILED/TERMINATED 状态 |

### PulseAudioAccess

| 方法 | 说明 |
|------|------|
| `new(change_signal, alsa_audio) -> Arc<Mutex<Self>>` | 初始化：创建 threaded_mainloop → pa_context_new_with_proplist → 设置状态回调 → context_connect → 启动主循环 → 等待 READY |
| `context_state_callback(c, pulse_ptr)` | C 回调：跟踪上下文连接状态（READY/FAILED），信号通知 |
| `subscribe_callback(c, event_bits, index, pulse_ptr)` | C 回调：设备变化订阅通知 |
| `sink_info_callback(ctx, info, eol, query_ptr)` | C 回调：收集音频输出设备信息 |
| `source_info_callback(ctx, info, eol, query_ptr)` | C 回调：收集音频源设备信息 |
| `server_info_callback(ctx, info, query_ptr)` | C 回调：获取默认 sink/source 名称 |
| `get_updated_descs() -> Vec<AudioDeviceDesc>` | 重新枚举设备：并行请求 sink_list + source_list + server_info，等待所有操作完成 |
| `use_audio_inputs(devices)` | 管理输入流：移除不在列表中的旧流，为新增设备创建 `PulseInputStream` |
| `use_audio_outputs(devices)` | 管理输出流：移除不在列表中的旧流，为新增设备创建 `PulseOutputStream`；失败设备加入 `failed_devices` 并触发信号 |

## 实现细节

### 音频参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 采样格式 | `PA_SAMPLE_FLOAT32LE` | 32 位浮点小端 |
| 采样率 | 48000 Hz | |
| 声道数 | 2 | 立体声 |
| 缓冲区帧数 | 256 | 每帧样本数 |
| 缓冲区属性 | `maxlength=u32::MAX, tlength=2048` | 大缓冲区减少欠载 |

### 流管理流程

```
PulseAudioAccess::new()
  ├ pa_threaded_mainloop_new()
  ├ pa_context_new_with_proplist()
  ├ 启动主循环线程
  └ 等待 PA_CONTEXT_READY

use_audio_inputs([devices])
  ├ 清理旧流（不在新列表中的）
  └ 为每个新设备：
     ├ pa_stream_new("makepad input stream")
     ├ pa_stream_set_read_callback()
     ├ pa_stream_connect_record()
     └ 等待 PA_STREAM_READY → 信号释放

use_audio_outputs([devices])
  ├ 清理旧流
  └ 为每个新设备：
     ├ pa_stream_new("makepad output stream")
     ├ pa_stream_connect_playback() [CORKED]
     ├ 等待 PA_STREAM_READY
     ├ pa_stream_set_write_callback()
     └ pa_stream_cork(0) → 开始播放
```

### 音频数据流

**输入 (Recording)**:
`pa_stream_peek` → `AudioBuffer::copy_from_interleaved(2, interleaved_f32)` → `input_fn(info, &buffer)`

**输出 (Playback)**:
`pa_stream_begin_write` → `output_fn(info, &mut buffer)` → `AudioBuffer::copy_to_interleaved(interleaved_f32)` → `pa_stream_write`

- 输入/输出均使用 interleaved float32 格式（L,R,L,R,...）
- 首次写入使用 `PA_SEEK_RELATIVE_ON_READ` 确保正确初始化
- 操作均在线程主循环锁保护下执行

### 设备枚举

`get_updated_descs()` 同时发起三个异步操作：
1. `pa_context_get_sink_info_list` — 音频输出设备
2. `pa_context_get_source_info_list` — 音频源设备（输入）
3. `pa_context_get_server_info` — 默认设备名称

等待所有操作完成后，生成 `AudioDeviceDesc` 列表，设备名称前缀 `[Pulse Audio]`。
