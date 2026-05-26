# `media_api.rs` — 平台媒体 API 抽象层

## 文件定位

该文件定义了 `CxMediaApi` trait，是 Makepad 框架中**媒体硬件抽象层（HAL）的核心接口**。它向上层 UI/应用代码提供统一的音频、MIDI、视频输入/输出/编码/解码 API，向下由各个平台后端（macOS、iOS、Android、Windows、Linux、Web）分别实现。整个 trait 位于 `crate::Cx` 的扩展中，与 `event`、`audio`、`midi`、`video` 模块深度协作。

---

## `CxMediaApi` trait

### `fn midi_input(&mut self) -> MidiInput`
返回当前 MIDI 输入端口集合的快照。平台后端在每个事件循环周期中更新该数据，反映物理 MIDI 端口的热插拔状态。内部通过 `MidiInput` 数据结构枚举所有已发现并启用的输入端口。

### `fn midi_output(&mut self) -> MidiOutput`
同理返回 MIDI 输出端口快照。用于应用层发现可用输出目标。

### `fn midi_reset(&mut self)`
完全重置 MIDI 子系统：关闭所有打开的端口、清空缓冲区、重新枚举设备。在设备热插拔或连接异常后调用，确保状态一致性。

### `fn use_midi_inputs(&mut self, ports: &[MidiPortId])`
声明应用层需要使用的 MIDI 输入端口列表。后端根据该列表打开对应端口并开始接收数据。未列出的端口会被静默忽略，这是一种**按需激活**设计，可降低功耗和资源占用。

### `fn use_midi_outputs(&mut self, ports: &[MidiPortId])`
声明需要使用的 MIDI 输出端口列表，逻辑同上。

### `fn use_audio_inputs(&mut self, devices: &[AudioDeviceId])`
声明要激活的音频输入设备。后端据此打开设备、建立音频捕获回调，并开始向应用层投递 `AudioInputFn` 数据。参数为 `AudioDeviceId` 切片，允许同时启用多个设备。

### `fn use_audio_outputs(&mut self, devices: &[AudioDeviceId])`
声明要激活的音频输出设备列表。后端为每个设备创建独立的音频渲染回调线程/上下文。

### `fn audio_output<F>(&mut self, index: usize, f: F)`
泛型包装方法：接收一个闭包 `f`，签名为 `FnMut(AudioInfo, &mut AudioBuffer) + Send + 'static`。内部将 `f` 装箱后转发给 `audio_output_box`。`index` 对应 `use_audio_outputs` 中设备的索引。闭包可在实时音频线程中调用，必须注意实时安全约束。

### `fn audio_input<F>(&mut self, index: usize, f: F)`
与 `audio_output` 对称，但闭包接收 `&AudioBuffer`（只读），用于处理从麦克风等设备捕获的 PCM 数据。

### `fn audio_output_box(&mut self, index: usize, f: AudioOutputFn)`
`AudioOutputFn` 是 `Box<dyn FnMut(AudioInfo, &mut AudioBuffer) + Send + 'static>` 的类型别名。各平台实现通过系统音频 API（CoreAudio、AAudio、OpenSL ES、WASAPI）建立回调，在音频中断中调用此闭包填充缓冲区。

### `fn audio_input_box(&mut self, index: usize, f: AudioInputFn)`
与 `audio_output_box` 对称，处理输入回调。后端在每次音频输入缓冲区就绪时调用 `f` 传递数据和元信息。

### `fn video_input<F>(&mut self, index: usize, f: F)`
泛型包装：为指定视频输入设备注册帧回调。闭包接收 `VideoBufferRef`，包含平台原生视频帧的引用。用于从摄像头或屏幕捕获获取帧数据。内部装箱后调用 `video_input_box`。

### `fn video_input_box(&mut self, index: usize, f: VideoInputFn)`
平台后端实现此方法以注册视频帧回调。后端在每帧解码后调用回调，传递 `VideoBufferRef`。实现因平台而异（macOS 用 AVFoundation 的 `captureOutput`，Android 用 `ImageReader.OnImageAvailableListener`）。

### `fn camera_frame_input<F>(&mut self, index: usize, f: F)`
高层次的摄像头帧输入 API。闭包接收 `CameraFrameRef<'a>`，它携带了比 `VideoBufferRef` 更丰富的元数据（如时间戳、朝向、内参）。泛型包装后调用 `camera_frame_input_box`。

### `fn camera_frame_input_box(&mut self, _index: usize, _f: CameraFrameInputFn)`
平台可选的摄像头帧传输钩子。提供默认空实现（no-op），支持结构化的 `CameraFrameRef` 传输的后端才需要覆写此方法。这种**渐进式抽象**设计允许框架逐步引入更丰富的帧数据格式，而不破坏现有平台。

### `fn video_encoder_output<F>(&mut self, index: usize, config: VideoEncoderConfig, f: F)`
创建视频编码会话。参数 `config` 包含编解码器类型、比特率、帧率、分辨率等。闭包 `f` 在每帧编码完成后被调用，接收 `EncodedVideoPacketRef`。内部调用 `video_encoder_output_try` 并在失败时自动记录错误（`crate::error!`）。

### `fn video_encoder_output_try<F>(...) -> Result<(), VideoEncodeError>`
与 `video_encoder_output` 功能相同，但将错误以 `Result` 形式返回而非自动记录。方便需要自定义错误处理的调用者。内部调用 `video_encoder_output_box`。

### `fn video_encoder_output_box(...) -> Result<(), VideoEncodeError>`
实际编码会话创建接口。默认实现返回 `Err(VideoEncodeError::UnsupportedSource)`。支持编码的平台后端必须覆写此方法，创建平台编码器（VideoToolbox、MediaCodec、NVENC）并建立编码输出的回调管道。

### `fn video_encoder_push_frame(&mut self, _index: usize, _frame: CameraFrameRef<'_>)`
向编码器推送一帧待编码数据。默认空实现。调用者在获取到 `CameraFrameRef` 后（可能来自 `camera_frame_input`），将其转发到编码器实例。

### `fn video_encoder_capture_texture_frame(&mut self, _index: usize, _timestamp_ns: u64) -> Result<(), VideoEncodeError>`
从已配置的纹理源捕获一帧进行编码。**必须在渲染线程上调用**，因为某些后端需要渲染上下文访问权限。默认返回 `UnsupportedSource`。用于从 GPU 纹理（如游戏画面或 GL 渲染结果）直接编码视频的场景。

### `fn video_encoder_request_keyframe(&mut self, _index: usize) -> Result<(), VideoEncodeError>`
请求编码器生成一个关键帧（IDR 帧）。默认返回 `UnsupportedCodec`。在流切换、丢包恢复或 Seek 操作时需要此功能。后端通过平台编码器 API 强制执行关键帧请求。

### `fn video_decoder_start_box(...) -> Result<(), VideoDecodeError>`
启动视频解码会话。接收 `VideoDecoderConfig`（包含编码格式、分辨率、codec-specific data）和解码帧输出回调。默认返回 `UnsupportedCodec`。平台后端创建解码器实例（VideoToolbox、MediaCodec、FFmpeg）并建立解码帧输出管道。

### `fn video_decoder_push_packet(&mut self, _index: usize, _packet: VideoDecoderPacketRef<'_>) -> Result<(), VideoDecodeError>`
向解码器推送一个压缩包。默认返回 `DecoderNotStarted`。调用者在视频流数据到达时逐个包提交；解码器内部进行序列化解码。

### `fn video_decoder_stop(&mut self, _index: usize)`
停止并清理解码会话。释放解码器持有的 GPU/硬件资源。默认空实现。

### `fn video_capabilities(&self) -> VideoCapabilities`
查询本平台支持的视频编解码能力。返回 `VideoCapabilities`，包含支持的编码/解码器列表、最大分辨率、帧率、对齐要求等。默认返回空能力集，平台后端根据硬件能力覆写。

### `fn use_video_input(&mut self, devices: &[(VideoInputId, VideoFormatId)])`
声明要激活的视频输入设备及其首选格式。与音频的 `use_audio_inputs` 对称。后端根据传入的设备-格式对打开视频捕获会话。

---

## 设计要点

1. **分层抽象**: 媒体 API 按"声明需要→获取回调→处理数据"的流程组织，上层只需声明需求、注册回调，无需关注平台差异。
2. **泛型+装箱双路径**: `audio_output`（泛型）提供类型安全的人体工学接口；`audio_output_box`（装箱）是平台实现的稳定边界。两者通过默认方法桥接。
3. **渐进式默认**: `video_decoder_start_box`、`video_encoder_output_box` 等方法的默认实现返回 `Err`，只有具备对应能力的平台才需覆写，避免了空实现的心智负担。
4. **实时音频安全**: `AudioOutputFn` 和 `AudioInputFn` 必须在音频实时线程中被调用，标注了 `Send + 'static` 以确保可跨线程安全传递。
5. **纹理源编码**: `video_encoder_capture_texture_frame` 专门为 GPU 端编码路径设计，必须有渲染上下文访问，因此文档显式标注了线程要求，避免误用。
