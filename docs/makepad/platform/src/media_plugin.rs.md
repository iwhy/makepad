# `media_plugin.rs` — 媒体插件系统与 MSE 播放引擎

## 文件定位

该文件是 Makepad 框架**媒体插件架构的核心定义文件**。它提供了三个层次的抽象：(1) MSE（Media Source Extensions）播放引擎的数据结构和 trait；(2) 平台视频帧解码器 trait；(3) 高层 `MediaPlugin` 和 `MediaPlaybackSession` trait。整个系统通过全局注册表 `MEDIA_PLUGIN`（`OnceLock<Arc<dyn MediaPlugin>>`）由外部共享库或平台原生代码填充，实现了**插件化媒体处理**。与 `media_api.rs`（应用侧 API）和 `media_host.rs`（框架桥接）一同构成完整的媒体栈。

---

## MSE 播放引擎模块

### `MseAudioTrackInfo` / `MseVideoTrackInfo`
表示 MSE 初始化段中声明的音频/视频轨道的元数据。包含轨道 ID、编解码器名称字符串、采样率/通道数（音频）和宽高（视频），以及编解码器私有配置数据 `config: Vec<u8>`（如 H.264 的 AVCC 记录或 AAC 的 DecoderSpecificInfo）。这些信息从 fMP4 初始化段的 `moov` box 中解析得到。

### `MseInitMetadata`
包含整个 MSE 流的元信息：时长（毫秒）、所有音频和视频轨道的详细列表。此结构由 `MsePlaybackEngine::append_data` 在处理第一个初始化段时生成，通过 `MseEngineOutput.init` 传递出去。

### `MseDecodedAudioFrame`
单个解码后的音频帧。包含 `track_id`（多轨道解复用用）、`pts_ms` 时间戳、采样率和通道数，以及 `samples: Vec<f32>`（交错的 PCM 浮点样本）。应用层（`MediaPlaybackSession::fill_audio_output`）消费这些样本填充音频输出缓冲区。

### `MseDecodedFrame`
单个解码后的视频帧。包含 `track_id`、`pts_ms` 时间戳和 `yuv: YuvPlaneData`。`YuvPlaneData` 在非 WASM 平台上是具体的 YUV 平面数据结构（可能包含三个平面的指针/偏移和跨距），在 WASM 上是占位类型。该结构通过 `take_yuv_frame` 从播放会话中取出，上传到 GPU 纹理进行渲染。

### `MseEngineOutput`
`append_data` 和 `end_of_stream` 的批量输出。它一次性返回：新生成的初始化元数据、新解码的音频帧列表、新解码的视频帧列表、更新的缓冲时间范围、以及是否已到达流末尾的标志。这种**批量输出设计**减少了多次查询的调用开销。`Default` 派生允许在无输出时返回空结构。

### `MsePlaybackEngine` trait
推模型 MSE 播放引擎的核心接口。实现者拥有内部的 demux、解码编排器和时序策略。

#### `fn append_data(&mut self, data: &[u8]) -> Result<MseEngineOutput, String>`
推送 fMP4 数据片段（初始化段或媒体段）。引擎内部通过 MP4 解析器解析 `moof`/`mdat` box，提取并解码音视频样本。返回解码后的帧和数据元信息。这种**推模型设计**使得引擎可以在数据到达时立即处理，不需要单独的解码循环。

#### `fn end_of_stream(&mut self) -> Result<MseEngineOutput, String>`
信号流结束。引擎刷新所有解码缓冲区、排出尚未输出的帧。返回值中包含最后一批解码帧。调用后引擎进入已结束状态。

#### `fn remove(&mut self, start: f64, end: f64)`
移除指定时间段（秒）内的缓冲数据。用于释放内存或在 Seek 后丢弃旧缓冲。引擎可能调整内部 `SourceBuffer` 的范围追踪。

#### `fn buffered_ranges(&self) -> Vec<(f64, f64)>`
返回当前缓冲的连续时间范围列表。每个 `(start, end)` 表示一段可播放的连续数据。用于 UI 显示进度条或 `MediaEventBridge::emit_video_buffered_ranges` 事件发射。

#### `fn flush(&mut self)`
刷新解码器状态。在 Seek 操作后调用，清除所有中间解码缓冲区以备新数据。不释放已分配的编解码器。

#### `fn cleanup(&mut self)`
释放引擎持有的所有资源（解码器实例、缓冲区分配、线程）。销毁前调用。

---

## 平台视频帧解码器

### `FrameDecoderCodec`
当前仅支持 H.264。可扩展为枚举的 match 臂以支持 VP9、AV1、HEVC 等。

### `FrameDecoderConfig`
包含编解码器类型、codec-specific 配置数据（如 H.264 的 AVCC/Annex B 头部）、宽度和高度。

### `VideoFrameDecoder` trait
推模型视频帧解码器。与 `MsePlaybackEngine` 不同，它专注于单一视频轨道的解码，不涉及 demux 或音频。

#### `fn push_data(&mut self, data: &[u8], pts_ms: u64)`
推送压缩帧数据（H.264 的 Annex B NAL 单元流）。`pts_ms` 由调用者管理，引擎只负责解码。

#### `fn pull_frame(&mut self) -> Result<Option<MseDecodedFrame>, String>`
拉取已解码的帧。返回 `None` 表示尚无可用帧。这种 push/pull 分离允许引擎内部实现异步解码，调用者在主循环中定期轮询。

#### `fn flush(&mut self)`
清理解码器内部状态，通常为 Seek 或 EOS 场景准备。

---

## `MediaVideoEncoder` trait

### `fn push_frame(&self, frame: CameraFrameRef<'_>)`
接收一帧摄像头帧并送入编码器。`CameraFrameRef` 可来自摄像头捕获或 CPU 程序化生成的帧。接受 `&self`（不可变引用），允许内部使用 `Mutex`/`RefCell` 管理可变状态，从而支持多线程调用。

### `fn push_apple_pixel_buffer(...)`（仅 macOS/iOS）
接收原生 `CVPixelBufferRef`，避免不必要的像素格式转换。返回 `bool` 表示是否成功接收。默认返回 `false`。如果插件支持直接处理 Apple 像素缓冲区，可覆写此方法以提供零拷贝编码路径。

### `fn request_keyframe(&self) -> Result<(), VideoEncodeError>`
请求编码器强制生成下一个关键帧。用于流切换或错误恢复。

### `fn stop(&self)`
停止编码器并释放资源。同步方法，返回后编码器不再产生输出。

---

## `PlaybackPrepared`

携带播放会话就绪时的媒体元信息：宽高、时长（毫秒）、是否可 Seek、音视频轨道列表。通过构造器 `new` 创建，由 `MediaPlaybackSession::check_prepared` 以 `Option<Result<...>>` 形式返回。

---

## `MediaPlaybackSession` trait

这是媒体播放的核心控制接口，统一了原生播放和 MSE 自定义播放两种路径。

| 方法 | 功能 | 实现细节 |
|---|---|---|
| `check_prepared` | 检查会话是否就绪 | 返回 `None`（未就绪）、`Some(Ok(...))`（就绪，含元信息）或 `Some(Err(...))`（初始化失败） |
| `poll_frame` | 轮询是否有新视频帧 | 返回 `true` 表示有新帧，可通过 `take_yuv_frame` 获取。实现者内部检查解码器输出队列 |
| `take_yuv_frame` | 取走当前 YUV 帧数据 | 返回 `Some(YuvPlaneData)` 或 `None`。调用后内部帧被消费 |
| `check_eos` | 检查是否已到达流末尾 | `true` 表示播放结束且无更多数据 |
| `play` | 开始/继续播放 | 启动解码线程/提交解码请求 |
| `pause` | 暂停播放 | 暂停解码但保持缓冲 |
| `resume` | 从暂停中恢复 | 与 `play` 的区别在于不重置状态 |
| `is_playing` | 查询播放状态 | 原子布尔检查 |
| `seek_to` | 跳转到指定位置 | 刷新解码器、丢弃旧缓冲、从新的位置重新开始馈送数据 |
| `set_volume` | 设置音量 | 浮点值，0.0 = 静音，1.0 = 原始音量 |
| `current_position_ms` | 当前播放位置 | 基于音频时钟或视频 PTS 追踪 |
| `mute` / `unmute` | 静音/取消静音 | 与 `set_volume(0.0)` 等效但保留原始音量值 |
| `set_playback_rate` | 设置播放速率 | 速率 ≠ 1.0 时可能需要调整 pitch |
| `seekable_ranges` | 可 Seek 范围列表 | 从 `SourceBuffer` 或媒体容器 seek index 获取 |
| `buffered_ranges` | 缓冲范围列表 | 用于进度条 UI 显示 |
| `fill_audio_output` | 填充音频输出缓冲区 | 默认空实现；MSE 播放器从 `MseEngineOutput.audio_frames` 队列中提取 PCM 数据填充 |
| `is_active` | 会话是否仍在活动 | 用于资源清理判断 |
| `cleanup` | 释放所有资源 | 停止解码器、释放纹理、关闭文件句柄 |

---

## `MediaPlugin` trait

插件系统的主入口。所有方法都有默认实现，插件只需覆写自身支持的功能。

| 方法 | 默认行为 | 覆写场景 |
|---|---|---|
| `create_video_encoder` | 返回 `None` | 平台支持硬件编码时（VideoToolbox/MediaCodec/NVENC） |
| `create_playback_session` | 返回 `Err(UnsupportedCodec)` | 平台支持媒体播放时 |
| `video_capabilities` | 返回空能力集 | 需声明平台解码/编码能力时 |
| `on_android_h264_packet` | no-op | Android MediaCodec 编码器输出回调 |
| `on_android_h264_error` | no-op | Android 编码错误通知 |
| `create_mse_playback_engine` | 返回 `Err("MSE not supported")` | 支持 MSE 自定义播放时（如 Ruffle/webcodecs 后端） |
| `create_video_frame_decoder` | 返回 `Err("not available")` | 需要与 MSE 引擎配合使用硬件解码时 |

---

## 全局注册与辅助函数

### `static MEDIA_PLUGIN: OnceLock<Arc<dyn MediaPlugin>>`
使用 `OnceLock` 确保插件全局只注册一次。`Arc` 包装支持多线程访问。

### `fn register_media_plugin(plugin: Arc<dyn MediaPlugin>) -> bool`
将插件实例注册到全局。返回 `true` 表示注册成功（首次），`false` 表示已有插件注册（忽略）。典型的插件初始化流程：构造插件 → 包装为 `Arc` → 调用此函数。

### `fn media_plugin() -> Option<&'static Arc<dyn MediaPlugin>>`
安全获取已注册的插件引用。返回 `None` 表示无注册插件。所有媒体操作在调用前都应检查此值。

### `fn media_video_capabilities() -> VideoCapabilities`
便捷函数：获取已注册插件的视频能力。无插件时返回默认空能力。

### `fn merge_video_capabilities(base, extra) -> VideoCapabilities`
将两个 `VideoCapabilities` 合并。合并策略可总结为**OR 逻辑 + 首次填充**：
- 编解码器支持标志（`encode_hardware`、`decode_software` 等）：按位 OR
- 格式列表：去重合并
- 源类型和功能支持标志：OR
- 宽度/高度对齐、最大宽高、最大帧率、最大比特率：`base` 中为 `None` 的字段用 `extra` 的值填充（首次写入语义）
- 编解码器条目：按 `codec` 名称匹配后合并，不存在的编解码器直接追加

此函数在初始化时由调用者（如 `Audio/Video` 子系统）用来聚合平台能力与插件能力。

---

## 设计要点

1. **三层抽象**: MSE 引擎 → VideoFrameDecoder + MediaVideoEncoder → MediaPlaybackSession + MediaPlugin，从底层解码到高层播放控制逐层封装。
2. **推模型解码**: `MsePlaybackEngine::append_data` 采用 push 语义，数据到达即可处理；`VideoFrameDecoder` 则使用 push（`push_data`）/pull（`pull_frame`）分离，适应不同的线程模型。
3. **插件无关架构**: Makepad 本身不依赖任何媒体库。`MediaPlugin` trait 允许第三方（如 GStreamer 插件、FFmpeg 插件）以动态库形式注入，框架与插件之间仅有 trait 约束。
4. **全局注册 + 可选实现**: `OnceLock` 保证线程安全和一次性初始化；所有方法都有优雅的默认行为，注册空插件不会破坏系统。
5. **能力合并的 OR+首次填充策略**: `merge_video_capabilities` 在选择硬编码与平台报告能力之间取得平衡，确保既不丢失平台实际能力，也不覆盖用户显式设置的限制。
