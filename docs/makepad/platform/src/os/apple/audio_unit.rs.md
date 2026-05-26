# audio_unit.rs — CoreAudio AudioUnit 封装

**文件路径:** `platform/src/os/apple/audio_unit.rs`

**核心目的:** 对 Apple CoreAudio 的 AudioUnit API 进行高层 Rust 封装。提供音频输入/输出、设备枚举、格式协商、音频处理图构建和实时音频回调管理。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `AudioUnitWrapper` | AudioUnit 实例的 Rust 包装器，提供类型安全 API |
| `AudioUnitGraph` | `AUGraph`/`AudioComponentInstance` 音频处理图管理 |
| `AudioDeviceManager` | 音频设备管理器，枚举和选择输入/输出设备 |
| `AudioDeviceInfo` | 音频设备信息（ID、名称、通道、采样率等） |
| `AudioStreamDescription` | 音频流格式描述（采样率、位深、通道数、格式标志） |
| `AudioBufferListWrapper` | 音频缓冲区列表的安全包装 |
| `AudioRenderCallback` | 音频渲染回调类型别名 |
| `AudioUnitProperty` | AudioUnit 属性 ID 枚举 |
| `AudioUnitParameter` | AudioUnit 参数 ID 枚举 |
| `AudioComponentDescription` | 音频组件描述 |

**`AudioUnitWrapper` 关键方法:**
- `AudioUnitWrapper::new(component_desc)` — 根据组件描述创建 AudioUnit 实例
- `AudioUnitWrapper::set_input_format(desc)` / `get_input_format()` — 输入格式协商
- `AudioUnitWrapper::set_output_format(desc)` / `get_output_format()` — 输出格式协商
- `AudioUnitWrapper::set_input_callback(cb)` / `set_output_callback(cb)` — 设置音频处理回调
- `AudioUnitWrapper::set_property(prop, data)` — 通用属性设置
- `AudioUnitWrapper::get_property(prop)` — 通用属性获取
- `AudioUnitWrapper::initialize()` — 初始化 AudioUnit
- `AudioUnitWrapper::start()` / `stop()` — 启动/停止音频处理
- `AudioUnitWrapper::dispose()` — 销毁 AudioUnit 实例

**`AudioUnitGraph` 关键方法:**
- `AudioUnitGraph::new()` — 创建新的 AUGraph/音频处理图
- `AudioUnitGraph::add_node(desc)` — 添加音频处理节点
- `AudioUnitGraph::get_node_info(node)` — 获取节点 AudioUnit
- `AudioUnitGraph::connect(src_node, src_bus, dest_node, dest_bus)` — 连接节点
- `AudioUnitGraph::open()` — 打开处理图（验证连接）
- `AudioUnitGraph::initialize()` — 初始化处理图
- `AudioUnitGraph::start()` / `stop()` — 启动/停止图处理
- `AudioUnitGraph::is_running()` — 检查图是否正在运行

**`AudioDeviceManager` 关键方法:**
- `AudioDeviceManager::new()` — 创建设备管理器
- `AudioDeviceManager::get_input_devices()` — 获取输入设备列表
- `AudioDeviceManager::get_output_devices()` — 获取输出设备列表
- `AudioDeviceManager::get_default_input()` — 获取默认输入设备（麦克风）
- `AudioDeviceManager::get_default_output()` — 获取默认输出设备（扬声器/耳机）
- `AudioDeviceManager::get_device_info(id)` — 获取设备详细信息
- `AudioDeviceManager::set_default_input(id)` / `set_default_output(id)` — 设置默认设备

**`AudioDeviceInfo` 字段:**
- `id: AudioDeviceID` — CoreAudio 设备 ID
- `name: String` — 设备名称
- `manufacturer: String` — 制造商
- `input_channels: u32` — 输入通道数
- `output_channels: u32` — 输出通道数
- `sample_rates: Vec<f64>` — 支持的采样率
- `is_input: bool` — 是否为输入设备
- `is_output: bool` — 是否为输出设备
- `is_default_input: bool` / `is_default_output: bool` — 是否为默认设备

**音频处理回调:**
```rust
type AudioRenderCallback = Box<dyn FnMut(&mut AudioBufferListWrapper, &AudioTimeStamp) -> Result<(), AudioUnitError>>;
```
- 在实时音频线程中调用（注意：不能加锁、分配内存或进行 I/O）
- 回调参数：`AudioBufferListWrapper`（可修改的音频缓冲区）、`AudioTimeStamp`（当前时间戳）
- 返回 `Result<(), AudioUnitError>`

**常见 AudioUnit 组件:**
| 子类型 | 描述 |
|--------|------|
| `kAudioUnitSubType_HALOutput` | 硬件抽象层输出（音频输出） |
| `kAudioUnitSubType_RemoteIO` | iOS 远程 I/O（音频输入/输出） |
| `kAudioUnitSubType_VoiceProcessingIO` | 语音处理 I/O（回声消除） |
| `kAudioUnitSubType_AUConverter` | 格式转换器 |
| `kAudioUnitSubType_AUFilePlayer` | 文件播放器 |
| `kAudioUnitSubType_AUAudioFilePlayer` | 音频文件播放器 (macOS) |
| `kAudioUnitSubType_Mixer` | 多通道混音器 |
| `kAudioUnitSubType_Splitter` | 音频信号分配器 |
| `kAudioUnitSubType_MultiChannelPanner` | 多通道声像定位器 |

**实现细节:**
- FFI 调用通过 `msg_send!` 和 CoreAudio C API 混合实现
- `AudioBufferListWrapper` 提供安全的内存管理（分配/释放 `AudioBufferList`）
- 格式协商通过 `AudioUnitSetProperty`/`AudioUnitGetProperty` 配合 `kAudioUnitProperty_StreamFormat` 实现
- 回调通过 `AudioUnitSetProperty` 设置 `kAudioUnitProperty_SetRenderCallback` 或 `kAudioOutputUnitProperty_SetInputCallback`
- 设备枚举通过 `AudioHardwareServiceGetPropertyInfo` 和 `AudioHardwareServiceGetPropertyData` 实现
- 音频格式使用 `AudioStreamBasicDescription` 结构体
- 非交织浮点格式（`kAudioFormatFlagIsNonInterleaved | kAudioFormatFlagIsFloat`）优先
- 错误处理覆盖：格式不匹配、权限不足、设备不可用、回调超时
- 通知监听：通过 `AudioHardwareServiceAddPropertyListener` 监听设备变化

**平台集成:** macOS 和 iOS，使用 CoreAudio / AudioToolbox 框架
