# wasapi.rs — Windows Audio Session API 音频 I/O

**文件路径**: platform/src/os/windows/wasapi.rs (923行)
**核心用途**: 通过 Windows WASAPI 实现音频输入（麦克风）、输出（扬声器）和 loopback（系统音频捕获），支持多设备枚举和异步音频回调。

## 结构体

### `WasapiAccess`
```rust
pub struct WasapiAccess {
    change_signal: SignalToUI,
    pub change_listener: IMMNotificationClient,  // 设备变更通知
    pub audio_input_cb: [Arc<Mutex<Option<AudioInputFn>>>; MAX_AUDIO_DEVICE_INDEX],
    pub audio_output_cb: [Arc<Mutex<Option<AudioOutputFn>>>; MAX_AUDIO_DEVICE_INDEX],
    enumerator: IMMDeviceEnumerator,
    audio_inputs: Arc<Mutex<Vec<WasapiBaseRef>>>,
    audio_outputs: Arc<Mutex<Vec<WasapiBaseRef>>>,
    descs: Vec<AudioDeviceDesc>,
    failed_devices: Arc<Mutex<HashSet<AudioDeviceId>>>,
}
```

### `WasapiBase`
```rust
struct WasapiBase {
    device_id: AudioDeviceId,
    device: IMMDevice,
    frames: u32,
    event: HANDLE,         // 事件句柄（信号帧就绪）
    client: IAudioClient,
    channel_count: usize,
    audio_buffer: Option<AudioBuffer>,
}
```

### `WasapiBaseRef`
```rust
struct WasapiBaseRef {
    device_id: AudioDeviceId,
    is_terminated: bool,   // 终止标志
    event: HANDLE,         // 事件句柄（用于唤醒线程检查终止）
}
```

### `WasapiOutput`
```rust
pub struct WasapiOutput {
    base: WasapiBase,
    render_client: IAudioRenderClient,
}
```

### `WasapiInput`
```rust
pub struct WasapiInput {
    base: WasapiBase,
    capture_client: IAudioCaptureClient,
}
```

### `WasapiLoopback`
```rust
pub struct WasapiLoopback {
    base: WasapiBase,         // 通过 new_loopback 创建的基实例
    capture_client: IAudioCaptureClient,
}
```

### `WasapiAudioOutputBuffer`
```rust
pub struct WasapiAudioOutputBuffer {
    frame_count: usize,
    channel_count: usize,
    device_buffer: *mut f32,  // 指向 WASAPI 设备缓冲区的指针
    pub audio_buffer: AudioBuffer, // Makepad 的音频缓冲区
}
```

### `WasapiChangeListener`
通过 `implement_com!` 注册 `IMMNotificationClient`，监听音频设备变更。

## 关键函数

### `WasapiAccess::new`
1. 调用 `CoInitializeEx(None, COINIT_APARTMENTTHREADED)`
2. 创建 `MMDeviceEnumerator`
3. 注册 `WasapiChangeListener` 接收设备变更通知

### 设备枚举

| 方法 | 描述 |
|------|------|
| `get_updated_descs` | 枚举所有输入、输出和 loopback 设备 |
| `enumerate_devices` | 枚举指定类型 (`eRender`/`eCapture`) 的活跃设备，返回名称、ID、默认状态和声道数 |
| `enumerate_loopback_devices` | 枚举输出设备作为 loopback 输入（设备 ID 格式: `原ID_loopback`） |
| `get_device_channel_count` | 通过 `IAudioClient::GetMixFormat` 获取原生声道数 |

### 音频流管理

| 方法 | 描述 |
|------|------|
| `use_audio_inputs(devices)` | 启用指定输入设备，为每个设备在新线程中启动 `WasapiInput` 或 `WasapiLoopback` |
| `use_audio_outputs(devices)` | 启用指定输出设备，为每个设备在新线程中启动 `WasapiOutput` |
| `is_loopback_device(device_id)` | 检查设备是否为 loopback 类型 |

### `WasapiBase::new`（共享模式初始化）
1. `device.Activate::<IAudioClient3>(CLSCTX_ALL)`
2. 创建 float 格式的 `WAVEFORMATEXTENSIBLE`（48000Hz, 32-bit float）
3. `GetSharedModeEnginePeriod` 获取默认周期帧数
4. `InitializeSharedAudioStream` + `AUDCLNT_STREAMFLAGS_EVENTCALLBACK`
5. 创建事件句柄，启动音频流

### `WasapiBase::new_loopback`
1. `device.Activate::<IAudioClient>(CLSCTX_ALL)`
2. `GetDevicePeriod` 获取默认周期
3. 强制最小 20ms 缓冲区
4. 使用 `AUDCLNT_STREAMFLAGS_LOOPBACK | AUTOCONVERTPCM | SRC_DEFAULT_QUALITY` 标志初始化

### 音频线程流程

输入线程：
```
WaitForSingleObject(event) → capture_client.GetBuffer → copy_from_interleaved → ReleaseBuffer → callback
```

输出线程：
```
WaitForSingleObject(event) → GetCurrentPadding → render_client.GetBuffer → callback → copy_to_interleaved → ReleaseBuffer
```

### 辅助函数

- `elevate_audio_thread_priority`: 调用 `AvSetMmThreadCharacteristicsW("Pro Audio")` 提升音频线程优先级
- `new_float_waveformatextensible`: 创建 32-bit float PCM 格式的 `WAVEFORMATEXTENSIBLE`，根据声道数设置通道掩码

## WAVEFORMATEXTENSIBLE 配置
- 采样率: 48000 Hz
- 位深: 32-bit float
- 格式: `WAVE_FORMAT_EXTENSIBLE` + `KSDATAFORMAT_SUBTYPE_IEEE_FLOAT`
- 通道掩码: 基于声道数的位掩码（最多 18 声道）

## IMMNotificationClient_Impl

`WasapiChangeListener` 响应 `OnDeviceStateChanged`、`OnDeviceAdded`、`OnDeviceRemoved`、`OnDefaultDeviceChanged` 事件，均触发 `change_signal.set()`。

## 平台集成

- 通过 `CxWindowsMedia::wasapi()` 延迟初始化
- `Cx::handle_media_signals()` 检查变更信号触发 `Event::AudioDevices`
- `CxMediaApi` 桥接：`use_audio_inputs`、`use_audio_outputs`、`audio_input_box`、`audio_output_box`
