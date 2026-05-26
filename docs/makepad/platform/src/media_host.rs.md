# `media_host.rs` — 媒体纹理/事件/控制桥接层

## 文件定位

该文件定义了三个 trait：`MediaTextureBridge`、`MediaEventBridge`、`MediaControlBridge`，并在 `Cx`（Makepad 应用上下文）上提供了具体实现。它构成了**媒体子系统与框架核心之间的胶水层**，负责纹理分配、事件通知和周期性的媒体时钟驱动。与 `media_api.rs`（面向应用的上层 API）不同，`media_host.rs` 是媒体插件与框架内部调度器之间的低级集成接口。

依赖关系：`texture`（纹理池分配）、`event`（事件分发）、`Cx`（全局上下文）。

---

## `MediaTextureInfo`

一个简单结构体，携带纹理的宽度和高度。由 `texture_info` 方法返回，用于查询桥接纹理的实际尺寸。应用层（如视频渲染器）可通过此信息设置 UV 坐标或调整视口大小。

---

## `MediaTextureBridge` trait

### `fn alloc_yuv_texture(&mut self) -> Texture`
在 Makepad 的纹理池中分配一张 `VideoYuvPlane` 格式的纹理。YUV 平面纹理用于接收解码后的视频帧数据。调用纹理池的 `alloc` 方法，返回的 `Texture` 句柄可直接用于 GPU 渲染。

### `fn alloc_external_texture(&mut self) -> Texture`
分配一张 `VideoExternal` 格式的纹理。该格式用于平台特定的外部纹理（如 Android 的 `GL_TEXTURE_EXTERNAL_OES`、iOS 的 `CVPixelBuffer` 后端纹理）。这种纹理不能被 Makepad 直接填充像素，而是由平台媒体框架直接写入。

### `fn texture_info(&self, texture_id: TextureId) -> Option<MediaTextureInfo>`
通过 `TextureId` 查询已分配纹理的实际尺寸。实现逻辑：从 `self.textures.0.pool` 中获取纹理池条目，再从条目的 `alloc` 字段读取宽高。返回 `None` 表示纹理不存在或尚未分配。用于适配不同分辨率视频源时的动态尺寸查询。

---

## `Cx` 对 `MediaTextureBridge` 的实现

### `alloc_yuv_texture`
直接将调用委托给 `self.textures.alloc(TextureFormat::VideoYuvPlane)`。纹理池的 `alloc` 方法会查找或创建一个可复用的纹理槽，标记其为 YUV 平面格式。

### `alloc_external_texture`
委托给 `self.textures.alloc(TextureFormat::VideoExternal)`。外部纹理的具体创建逻辑由各平台渲染后端（Metal、Vulkan、OpenGL）的纹理池实现决定。

### `texture_info`
三步查找：`textures.0.pool.get(texture_id.0)` 拿到 `PoolItem` → `pool_item.item.alloc.as_ref()` 拿到分配信息 → 构造 `MediaTextureInfo`。使用 `Option` 链保证了在不存在的纹理上安全返回 `None`。

---

## `MediaEventBridge` trait

定义了七个视频事件发射方法，每个方法接收对应的事件结构体，通过 `Cx::call_event_handler` 注入 Makepad 的事件系统。

| 方法 | 对应 Event 变体 | 触发时机 |
|---|---|---|
| `emit_video_prepared` | `Event::VideoPlaybackPrepared` | 视频解码器就绪、元信息（宽高、时长）可获取时 |
| `emit_video_texture_updated` | `Event::VideoTextureUpdated` | 新帧已解码到目标纹理，纹理内容更新完成时 |
| `emit_video_completed` | `Event::VideoPlaybackCompleted` | 播放完成（正常结束或用户终止）时 |
| `emit_video_error` | `Event::VideoDecodingError` | 解码器遇到不可恢复错误时 |
| `emit_video_yuv_ready` | `Event::VideoYuvTexturesReady` | YUV 三平面纹理全部就绪、可渲染时 |
| `emit_video_seekable_ranges` | `Event::VideoSeekableRanges` | 可 Seek 范围发生变化（如直播流范围扩展）时 |
| `emit_video_buffered_ranges` | `Event::VideoBufferedRanges` | 缓冲范围发生变化（加载更多数据）时 |

所有方法都通过 `self.call_event_handler(&Event::Xxx(event))` 工作。`call_event_handler` 是 Cx 的核心事件分发方法，将事件推入当前应用的 `handle_event` 调用链。

---

## `MediaControlBridge` trait

### `fn media_tick(&mut self)`
媒体子系统周期性时钟回调。当前 Cx 实现为空方法。媒体插件可通过覆写此方法实现帧步进、音视频同步漂移校正、缓冲状态检查等周期性任务。此方法通常在应用主循环的每个帧中被调用一次。

### `fn media_handle_video_surface_update(&mut self, _video_id: LiveId)`
当特定视频表面的属性（如尺寸、旋转角度）发生变化时调用。默认空实现。接收 `LiveId` 以标识具体是哪个 `Video` widget 的表面需要更新。渲染器可在此方法中重新计算视口或重排纹理坐标。

---

## 设计要点

1. **关注点分离**: 三个 trait 分别管理纹理资源、事件通知和时钟控制，粒度精确且可独立演进。媒体插件只需实现需要的 trait，无需关注其他方面。
2. **统一的事件桥接**: 所有视频相关事件都通过 `MediaEventBridge` 集中发射，确保了事件类型的安全性和可追踪性。事件由框架的 `Event` 枚举统一管理和路由。
3. **零成本抽象**: 所有默认实现（如 `media_tick` 的空方法）在编译时被优化掉，不会产生运行时开销。
4. **纹理池复用**: `alloc_yuv_texture` 和 `alloc_external_texture` 使用 Makepad 的纹理池机制，支持纹理复用和自动垃圾回收，避免了高频分配/释放导致的 GPU 内存碎片。
