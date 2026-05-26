# alsa_audio.rs — ALSA 音频设备管理与音频输入输出

**文件路径**: `platform/src/os/linux/alsa_audio.rs` (503 行)
**核心功能**: 通过 ALSA（Advanced Linux Sound Architecture）API 实现音频输入/输出设备的枚举、打开、读写操作。

## 主要类型

### `AlsaAudioAccess`
音频访问的顶层管理器，包含：
- `audio_input_cb` / `audio_output_cb` — 每个设备索引的回调函数数组
- `audio_outputs` / `audio_inputs` — 当前活动的设备引用列表（线程安全）
- `device_descs` — 缓存的设备描述列表
- `failed_devices` — 已标记为失败的设备集合
- `change_signal` — 用于通知 UI 线程设备变化的信号

### `AlsaAudioDevice`
单个 ALSA PCM 设备句柄：
- `device_handle` — `snd_pcm_t*` 原生指针
- `channel_count` / `frame_count` — 声道数和帧数
- `interleaved` — 交错格式的 f32 缓冲区

### `AlsaAudioDeviceRef`
设备引用的轻量级标识：
- `device_id` — 设备标识
- `is_terminated` — 标记是否需要终止

### `AlsaError`
封装 ALSA API 错误的类型，包含错误消息字符串。

## 关键方法

### `AlsaAudioAccess::new(change_signal)`
创建音频访问实例，同时启动后台线程每秒轮询声卡数量变化，检测到变化时通过 `change_signal` 通知 UI。

### `AlsaAudioAccess::get_updated_descs()`
枚举所有 ALSA 声卡设备，通过 `snd_device_name_hint` 获取 PCM 设备列表。根据 IOID 区分输入/输出设备，并自动选择默认设备（优先 `plughw:`，次选 `dmix:`，最后选第一个可用设备）。

### `AlsaAudioAccess::use_audio_inputs(devices)`
启动指定设备的音频输入线程：
- 终止不再使用的设备
- 为新设备创建独立的捕获线程
- 每个线程循环调用 `read_input_buffer` 并通过回调传递数据

### `AlsaAudioAccess::use_audio_outputs(devices)`
类似 `use_audio_inputs`，但用于音频输出：
- 为新设备创建播放线程
- 每个线程循环调用回调获取数据，然后通过 `write_output_buffer` 写入 ALSA

### `AlsaAudioDevice::new(device_name, device_id, direction)`
打开 ALSA PCM 设备，配置硬件参数：
- 采样率：48000 Hz
- 格式：FLOAT_LE（32位浮点）
- 访问模式：交错（interleaved）
- 声道数：2
- 自动设置 period 和 buffer 大小

### `AlsaAudioDevice::write_output_buffer(buffer)`
将 `AudioBuffer` 交错化后通过 `snd_pcm_writei` 写入设备。处理 `EPIPE`（缓冲区欠载/过载）错误。

### `AlsaAudioDevice::read_input_buffer(buffer)`
通过 `snd_pcm_readi` 从设备读取数据，解交错后存入 `AudioBuffer`。

## 实现细节

- 使用 32 位浮点格式以避免整数转换开销
- 后台线程每秒轮询检测声卡热插拔
- 失败设备会被记录并通过 `change_signal` 通知 UI 重新枚举
- 默认设备选择策略：`plughw:` > `dmix:` > 任意设备
