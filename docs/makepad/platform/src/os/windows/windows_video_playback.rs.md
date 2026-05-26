# windows_video_playback.rs — Media Foundation 视频播放

**文件路径**: platform/src/os/windows/windows_video_playback.rs (924行)
**核心用途**: 通过 Windows Media Foundation 的 IMFMediaEngine 实现硬件加速视频解码和渲染，支持文件、网络流和内存数据源。

## Windows 视频播放架构

使用 `IMFMediaEngine`（Windows 平台的系统级播放器，等效于 macOS 的 AVPlayer 或 Linux 的 GStreamer playbin），处理音视频解码和 A/V 同步。

## GUID 定义

| GUID | 值 | 用途 |
|------|-----|------|
| `CLSID_MF_MEDIA_ENGINE_CLASS_FACTORY` | `B44392DA-499B-446B-A4CB-005FEAD0E6D5` | 引擎工厂类 |
| `IID_IMF_MEDIA_ENGINE_CLASS_FACTORY` | `4D645ACE-26AA-4688-9BE1-DF3516990B93` | 工厂接口 |
| `IID_IMF_MEDIA_ENGINE_NOTIFY` | `FEE7C112-E776-42B5-9BBF-0048524E2BD5` | 通知回调接口 |
| `MF_MEDIA_ENGINE_CALLBACK` | `C60381B8-83A4-41F8-A3D0-DE05076849A9` | 属性键：通知回调 |
| `MF_MEDIA_ENGINE_DXGI_MANAGER` | `065702DA-1094-486D-8617-EE7CC4EE4648` | 属性键：DXGI 管理器 |
| `MF_MEDIA_ENGINE_VIDEO_OUTPUT_FORMAT` | `5066893C-8CF9-42BC-8B8A-472212E52726` | 属性键：输出格式 |

## 事件常量

| 事件 | 值 | 含义 |
|------|-----|------|
| `ME_EVENT_ERROR` | 5 | 播放错误 |
| `ME_EVENT_CANPLAY` | 14 | 可以播放 |
| `ME_EVENT_ENDED` | 19 | 播放结束 |
| `ME_EVENT_FORMATCHANGE` | 1000 | 媒体格式变更 |

## COM Vtable 定义

### `IMFMediaEngineVtbl`
完整的 COM vtable（3 个 IUnknown 方法 + 38 个 IMFMediaEngine 方法），包括：
- 播放控制：`Play`、`Pause`、`Load`
- 时间管理：`GetCurrentTime`、`SetCurrentTime`、`GetDuration`
- 属性：`GetVolume`、`SetVolume`、`GetMuted`、`SetMuted`、`GetLoop`、`SetLoop`
- 查询：`HasVideo`、`HasAudio`、`GetNativeVideoSize`、`GetVideoAspectRatio`
- 帧传输：`TransferVideoFrame`、`OnVideoStreamTick`
- 生命周期：`Shutdown`

### `IMFMediaEngineClassFactoryVtbl`
工厂类 vtable，含 `CreateInstance` 方法创建引擎。

### `IMFDXGIDeviceManagerVtbl`
DXGI 设备管理器 vtable，含 `ResetDevice`、`OpenDeviceHandle`、`GetVideoService` 等。

## 辅助结构体

### `MFVideoNormalizedRect`
```rust
struct MFVideoNormalizedRect { left, top, right, bottom: f32 }
```

### `RECT`
简化 Win32 RECT（left, top, right, bottom: i32）。

### `MFARGB`
```rust
struct MFARGB { blue, green, red, alpha: u8 }
```

## 通知回调 (`MediaEngineNotify`)

手动实现的 COM 对象（无宏），接收 `IMFMediaEngineNotify::EventNotify`：
- 引用计数：`AtomicU32`
- 事件队列：`Mutex<Vec<u32>>`
- 方法 `drain_events`: 取出并清空事件列表

## 结构体

### `WindowsVideoPlayer`
```rust
pub struct WindowsVideoPlayer {
    engine: *mut c_void,
    notify: *mut MediaEngineNotify,
    dxgi_manager: *mut c_void,
    d3d11_device: ID3D11Device,
    render_texture: Option<ID3D11Texture2D>,
    render_srv: Option<ID3D11ShaderResourceView>,
    pub(crate) video_id: LiveId,
    texture_id: TextureId,
    is_prepared: bool,
    prepare_notified: bool,
    prepare_error: Option<String>,
    is_eos: bool,
    eos_notified: bool,
    autoplay: bool,
    video_width: u32,
    video_height: u32,
    temp_file_path: Option<PathBuf>,
}
```

## 关键函数

### `create_engine_on_mta`（在 MTA 线程创建引擎）

`IMFMediaEngine` 需要 MTA（多线程单元），Makepad 的 UI 线程是 STA，因此在独立线程上创建：

1. `CoInitializeEx(MTA)` + `MFStartup`
2. 创建 `DXGIDeviceManager`，用 `ResetDevice` 绑定 D3D11 设备
3. 启用 `ID3D10Multithread::SetMultithreadProtected(true)`（允许多线程共享设备）
4. 设置属性：
   - `MF_MEDIA_ENGINE_CALLBACK`: 通知回调
   - `MF_MEDIA_ENGINE_DXGI_MANAGER`: DXGI 管理器
   - `MF_MEDIA_ENGINE_VIDEO_OUTPUT_FORMAT`: B8G8R8A8_UNORM
5. 通过 `CoCreateInstance` 创建引擎工厂，调用 `CreateInstance`
6. 设置循环模式，调用 `SysAllocString` + `SetSource` 设置媒体 URL
7. 返回引擎和 DXGI 管理器指针

### `WindowsVideoPlayer::new`
1. 根据 `VideoSource` 生成宽字符 URL：
   - `Network`: 直接使用 URL
   - `Filesystem`: 转换为 `file:///` 格式
   - `InMemory`: 写入临时文件再转换为 `file:///` URL
2. 创建 `MediaEngineNotify`
3. 在 MTA 线程上创建引擎
4. 返回 `WindowsVideoPlayer` 实例

### `can_play_type`
返回 Windows Media Foundation 支持的 MIME 类型：
- `video/mp4`、`video/x-m4v`: "probably"
- `audio/mp4`、`audio/x-m4a`、`audio/mpeg`、`audio/wav`: "probably"
- `video/webm`、`audio/webm`: "maybe"
- 其他视频/音频: "maybe"
- 未知: ""

### `process_events`
处理 `MediaEngineNotify` 事件：
- `ME_EVENT_CANPLAY`: 标记 `is_prepared = true`
- `ME_EVENT_ENDED`: 标记 `is_eos = true`
- `ME_EVENT_FORMATCHANGE`: 更新视频尺寸，重置渲染纹理
- `ME_EVENT_ERROR`: 记录错误信息

### `ensure_render_texture`
创建 D3D11 纹理和着色器资源视图：
- 格式: `DXGI_FORMAT_B8G8R8A8_UNORM`
- 标志: `D3D11_BIND_RENDER_TARGET | D3D11_BIND_SHADER_RESOURCE`

### `poll_frame`
视频帧拉取和纹理更新：
1. 调用 `OnVideoStreamTick` 检查新帧
2. `TransferVideoFrame` 将解码帧复制到 D3D11 纹理
3. 更新 `CxTexturePool` 中的纹理，设置格式为 `TextureFormat::VideoExternal`

### 播放控制
| 方法 | 行为 |
|------|------|
| `play` | 重置 EOS 标记，调用 `Play` |
| `pause` | 调用 `Pause` |
| `resume` | 调用 `Play` |
| `mute` / `unmute` | 调用 `SetMuted(1/0)` |
| `seek_to` | 计算秒数，调用 `SetCurrentTime`，重置 EOS |
| `current_position_ms` | `GetCurrentTime * 1000` |
| `is_playing` | `!IsPaused && !IsEnded` |

### `cleanup` / `Drop`
释放所有资源：`Shutdown` → `com_release` → `notify_release` → `com_release(dxgi_manager)` → 清除纹理 → 删除临时文件。

## 平台集成

- `WindowsUnifiedVideoPlayer` 是统一入口（自动回退到软件解码）
- 在 `windows.rs` 中通过 `Paint` 事件循环轮询帧
- 使用 `CxTexturePool::VideoExternal` 纹理格式保持引擎纹理引用
- 硬件解码路径直接将 Media Foundation 纹理复制到渲染纹理
