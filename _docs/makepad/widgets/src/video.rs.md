# video.rs — 视频播放器组件

## 概述
`Video` 组件使用 Makepad 的媒体管线播放视频文件。支持播放/暂停控制、循环播放、音量调节、进度跳转、自动播放和画面缩放模式（`Fit`/`Fill`）。通过 `DrawVideo` 绘制器在 GPU 上渲染视频帧。

## 核心结构

### Video
- **`draw_video: DrawVideo`**：视频帧绘制器。
- **`video_player: Option<VideoPlayer>`**：媒体播放器实例。
- **`video_data: Option<ScriptHandleRef>`**：视频文件的资源句柄。
- **`mode: DrawVideoMode`**：缩放模式（`Fit`/`Fill`）。
- **`looping: bool`**：是否循环播放。
- **`autoplay: bool`**：是否自动播放。
- **`volume: f32`**：音量（0.0-1.0）。

### DrawVideo
视频帧绘制器，包含 `#[live] frame: DrawVideoFrame`（当前帧纹理引用）、`mode: DrawVideoMode`（缩放模式）和 `gpu: GpuTexture`（GPU 纹理）。

### DrawVideoMode
枚举：`Fit`（保持宽高比，黑边填充）、`Fill`（拉伸充满画面，可能裁剪）。

### VideoPlayer
Makepad 媒体层的抽象，提供跨平台的视频解码和播放。包含状态管理（播放/暂停/结束/错误）、帧推进、音量控制和播放进度查询。

## 核心方法

### Widget 实现

**`draw_walk`**：调用 `update_player` 推进视频播放器，检查是否需要加载视频数据。使用 `try_get_frame` 获取最新帧纹理并传递给 DrawVideo 绘制器。

**`handle_event`**：处理 `ScriptHandleLoaded` 事件（视频文件加载完成）、`Event::Window->Event::VideoFrame` 事件（新视频帧可用，触发重绘）、鼠标/触摸交互（点击切换播放/暂停）。

### 视频控制

**`play`/`pause`/`toggle_playback`**：控制播放状态。`toggle_playback` 在播放中暂停，暂停中恢复。

**`set_looping`/`is_looping`**：设置/查询循环播放。

**`set_volume`/`volume`**：设置/查询音量。

**`set_progress`/`progress`/`duration`**：设置/查询播放进度（0.0-1.0）和总时长（秒）。

**`is_playing`**：查询是否正在播放。

**`seek_to`**：跳转到指定时间位置（秒）。

### 资源管理

**`update_player`**：检查 `video_data`，如果尚未加载则请求资源加载。加载完成后创建 `VideoPlayer`，设置循环和自动播放。

**`try_get_frame`**：从 `VideoPlayer` 获取最新视频帧，将其纹理数据拷贝到 `DrawVideo` 的 `gpu` 纹理中，更新 `frame` 引用。

### VideoPlayer 接口

`VideoPlayer` 提供跨平台视频解码：
- `new(cx, path)`：创建播放器实例。
- `update`：推进解码（返回是否有新帧）。
- `try_get_frame`：获取当前帧纹理。
- `play`/`pause`/`stop`：播放控制。
- `seek(time)`：跳转。
- `set_volume(vol)`/`set_looping(loop)`：参数设置。
- `duration`/`current_time`：状态查询。
- `is_playing`/`is_finished`/`has_error`：状态查询。
- `drop`：释放资源和解码器。

### VideoRef

`VideoRef` 提供所有播放控制方法的安全 `borrow`/`borrow_mut` 委托版本。
