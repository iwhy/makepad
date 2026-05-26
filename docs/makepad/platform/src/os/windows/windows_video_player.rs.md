# windows_video_player.rs — 统一视频播放器（原生+软件解码回退）

**文件路径**: platform/src/os/windows/windows_video_player.rs (342行)
**核心用途**: 提供统一的视频播放接口，优先使用 Media Foundation 硬件解码，失败时自动回退到软件解码器，并支持 YUV 纹理上传到 D3D11。

## 枚举

### `WindowsPlayerMode`
```rust
enum WindowsPlayerMode {
    Native(WindowsVideoPlayer),         // IMFMediaEngine 硬件解码
    Software(PlaybackSessionHandle),    // 软件解码回退
}
```

## 结构体

### `WindowsUnifiedVideoPlayer`
```rust
pub struct WindowsUnifiedVideoPlayer {
    pub(crate) video_id: LiveId,
    texture_id: TextureId,               // 主视频纹理 ID（RGBA/外部）
    tex_y_id: TextureId,                 // Y 平面纹理 ID
    tex_u_id: TextureId,                 // U 平面纹理 ID
    tex_v_id: TextureId,                 // V 平面纹理 ID
    yuv_matrix: f32,                     // YUV→RGB 色彩空间矩阵
    d3d11_device: ID3D11Device,
    source: VideoSource,
    autoplay: bool,
    is_looping: bool,
    mode: WindowsPlayerMode,
}
```

## 关键函数

### `new` — 创建统一播放器
1. 检查 `MAKEPAD_FORCE_SOFTWARE_VIDEO` 环境变量
2. 如果设置了环境变量 → 强制使用软件解码器
3. 否则尝试创建 `WindowsVideoPlayer`（硬件解码）
4. 如果硬件解码失败 → 回退到 `PlaybackSessionHandle`（软件解码）
5. 存储 Y/U/V 三个纹理 ID（用于软件解码的 YUV 平面）

### `switch_to_software` — 运行时回退
在 `check_prepared` 返回错误时自动调用，替换 `WindowsPlayerMode::Native` 为 `Software`。

### 帧轮询

#### `poll_frame`
- **Native 模式**: 调用 `player.poll_frame(textures)`，使用 `TextureFormat::VideoExternal`
- **Software 模式**:
  1. `player.poll_frame()` 检查新帧
  2. `player.take_yuv_frame()` 获取 YUV 平面数据
  3. 记录 `yuv_matrix`
  4. `upload_yuv_to_d3d11` 将三个平面上传到 D3D11 纹理

### `upload_yuv_to_d3d11`

将软件解码的 YUV 平面上传到 D3D11 纹理：
1. `chroma_size` 计算 UV 平面尺寸
2. 分别上传 Y、U、V 平面

### `upload_r8_plane_to_d3d11`

单平面上传：
1. 创建 `D3D11_TEXTURE2D_DESC`（`DXGI_FORMAT_R8_UNORM`，`D3D11_BIND_SHADER_RESOURCE`）
2. `CreateTexture2D` 传入 `D3D11_SUBRESOURCE_DATA`（包含数据指针和宽度）
3. 创建 `ID3D11ShaderResourceView`
4. 更新 `CxTexturePool` 中对应纹理，格式为 `TextureFormat::VideoYuvPlane`

### 播放控制代理

所有控制方法（`play`、`pause`、`resume`、`mute`、`unmute`、`seek_to`、`set_volume`、`set_playback_rate`、`cleanup`、`check_prepared`、`check_eos`、`is_playing`、`current_position_ms`）均根据当前模式代理到相应的底层播放器。

## 平台集成

- 在 `windows.rs` 的 `CxOsOp::PrepareVideoPlayback` 中创建
- 在 `Paint` 事件中每帧调用 `poll_frame` 和 `check_eos`
- 通过 `is_software_mode` 和 `yuv_matrix` 提供给上层事件
- 纹理格式由调用者决定：YUV 平面纹理 (`tex_y_id/tex_u_id/tex_v_id`) 在创建时分配
