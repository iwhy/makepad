# audio_tap.rs — 系统音频捕获

**文件路径:** `platform/src/os/apple/audio_tap.rs`

**核心目的:** 使用 CoreAudio 的 `AudioTap` API 捕获系统音频输出（扬声器渲染的音频）。支持实时音频处理、录制和音频分析。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `AudioTapManager` | 音频 Tap 管理器，管理系统音频捕获的完整生命周期 |
| `AudioTapStream` | 单个音频 Tap 流，包含音频数据处理管道 |
| `AudioTapConfig` | Tap 配置（采样率、通道数、格式、处理回调） |
| `AudioTapBuffer` | 捕获的音频缓冲区，包含 PCM 数据和元数据 |
| `ProcessedAudioBuffer` | 处理后的音频缓冲区，支持格式转换和重采样 |
| `AudioTapFlags` | Tap 标志（渲染输入、渲染输出、系统输出等） |

**关键方法:**
- `AudioTapManager::new()` — 创建 AudioTapManager 实例
- `AudioTapManager::start_tap(config)` — 启动指定配置的音频 Tap：
  - 创建 `AVAudioEngine` 和音频处理图
  - 安装 Tap 到目标音频节点
  - 注册音频处理回调
- `AudioTapManager::stop_tap()` — 停止音频 Tap
- `AudioTapManager::process_buffer(cb)` — 处理音频缓冲区回调
- `AudioTapManager::get_running_processes()` — 获取所有运行中的音频进程列表

**`AudioTapConfig` 字段:**
- `sample_rate: f64` — 目标采样率（默认 44100 Hz）
- `channel_count: u32` — 通道数（默认 2 立体声）
- `bit_depth: u32` — 位深度（16 或 32 位浮点）
- `format_flags: AudioFormatFlags` — CoreAudio 格式标志
- `processing_cb: Box<dyn FnMut(&AudioTapBuffer)>` — 音频处理回调

**`AudioTapBuffer` 字段:**
- `data: Vec<u8>` — 原始 PCM 音频数据
- `sample_rate: f64` — 实际采样率
- `channel_count: u32` — 实际通道数
- `frame_count: u32` — 帧数
- `timestamp: f64` — 捕获时间戳
- `peak_level: f32` — 峰值电平（用于音量表）

**实现细节:**
- 使用 `AVAudioEngine` 创建音频处理图
- 通过 `AVAudioNode` 的 `installTapOnBus:bufferSize:format:block:` 安装音频 Tap
- Tap 位置在音频引擎的输出节点（渲染后的音频）
- 使用 `AudioConverterNew` 进行格式转换（如 Float32→Int16）
- 缓冲区分块处理以适应实时性要求
- 电平计算使用 `vDSP_vflt32` 或简单平方根峰值检测
- `get_running_processes()` 使用 `AudioHardwareService` 枚举系统音频进程
- 错误处理：格式不支持、权限不足、Tap 安装失败

**平台集成:** macOS 专用（iOS 音频 Tap API 限制），使用 AVFoundation 和 CoreAudio
