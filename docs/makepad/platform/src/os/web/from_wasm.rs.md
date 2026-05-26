# from_wasm.rs — Rust→JS (FromWasm) 消息类型定义

**文件路径**: `/home/ubuntu/_github/makepad/platform/src/os/web/from_wasm.rs` (408 行)
**核心作用**: 定义所有从 Rust 发送到 JavaScript 的消息结构体，涵盖浏览器操作、WebGL、音频、MIDI、视频播放等领域。

## 消息结构体分类

### 浏览器/窗口操作

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `FromWasmStartTimer` | `repeats`, `timer_id`, `interval` | 启动定时器 |
| `FromWasmStopTimer` | `timer_id` | 停止定时器 |
| `FromWasmFullScreen` | — | 请求全屏 |
| `FromWasmNormalScreen` | — | 退出全屏 |
| `FromWasmRequestAnimationFrame` | — | 请求下一动画帧 |
| `FromWasmSetDocumentTitle` | `title` | 设置文档标题 |
| `FromWasmSetMouseCursor` | `web_cursor: u32` | 设置鼠标样式（映射 `MouseCursor` 枚举到 0-25 数值） |
| `FromWasmTextCopyResponse` | `response` | 剪贴板复制响应 |
| `FromWasmOpenUrl` | `url`, `in_place` | 打开 URL |
| `FromWasmBrowserUpdateUrl` | `url`, `replace` | 更新浏览器 URL |
| `FromWasmBrowserHistoryGo` | `delta` | 浏览器历史导航 |
| `FromWasmShowTextIME` | `x`, `y` | 显示输入法 |
| `FromWasmHideTextIME` | — | 隐藏输入法 |

### HTTP 网络

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `FromWasmHTTPRequest` | `request_id_lo/hi`, `metadata_id_lo/hi`, `url`, `method`, `headers`, `body` | 发起 HTTP 请求 |
| `FromWasmCancelHTTPRequest` | `request_id_lo/hi` | 取消 HTTP 请求 |

### 权限

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `FromWasmCheckPermission` | `permission`, `request_id` | 检查权限状态 |
| `FromWasmRequestPermission` | `permission`, `request_id` | 请求权限 |

### WebGL 图形

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `FromWasmCompileWebGLShader` | `shader_id`, `vertex`, `pixel`, `geometry_slots`, `instance_slots`, `textures` | 编译 WebGL 着色器 |
| `FromWasmAllocArrayBuffer` | `buffer_id`, `data: WasmPtrF32` | 分配顶点数组缓冲区 |
| `FromWasmAllocIndexBuffer` | `buffer_id`, `data: WasmPtrU32` | 分配索引缓冲区 |
| `FromWasmAllocVao` | `vao_id`, `shader_id`, `geom_ib_id`, `geom_vb_id`, `inst_vb_id` | 分配顶点数组对象 |
| `FromWasmAllocTextureImage2D_BGRAu8_32` | `texture_id`, `width`, `height`, `data: WasmPtrU32` | 分配 BGRA u8 纹理 |
| `FromWasmAllocTextureImage2D_Ru8` | `texture_id`, `width`, `height`, `data: WasmPtrU8` | 分配 R u8 纹理 |
| `FromWasmAllocTextureImage2D_RGBAf32` | `texture_id`, `width`, `height`, `data: WasmPtrF32` | 分配 RGBA f32 纹理 |
| `FromWasmAllocTextureCube_BGRAu8_32` | `texture_id`, `width`, `height`, `data: WasmPtrU32` | 分配 CubeMap BGRA 纹理 |
| `FromWasmBeginRenderTexture` | `pass_id`, `width`, `height`, `color_targets: [WColorTarget; 1]`, `depth_target: WDepthTarget` | 开始渲染到纹理 |
| `FromWasmBeginRenderCanvas` | `clear_color`, `clear_depth` | 开始渲染到画布 |
| `FromWasmSetDefaultDepthAndBlendMode` | — | 设置默认深度/混合模式 |
| `FromWasmDrawCall` | `vao_id`, `shader_id`, 深度/剔除标志, 6 个 uniforms 缓冲区指针, 纹理数组 | 执行一次绘制调用 |

### 辅助类型

| 类型 | 字段 | 用途 |
|------|------|------|
| `WTextureInput` | `ty` ("sampler2D"/"samplerCube"), `name` | 着色器纹理输入描述 |
| `WColor` | `r`, `g`, `b`, `a: f32` | 颜色值（可 `Into<WColor>` for `Vec4f`） |
| `WColorTarget` | `texture_id`, `init_only`, `clear_color` | 渲染颜色目标 |
| `WDepthTarget` | `texture_id`, `init_only`, `clear_depth` | 渲染深度目标 |

纹理类型映射（在 `to_from_wasm_texture_input` 中）：
- `TextureCube` / `TextureCubeArray` → `"samplerCube"`
- 其他 → `"sampler2D"`

鼠标光标映射 (`FromWasmSetMouseCursor::new`) 将 `MouseCursor` 枚举 26 个变体映射到 0-25 的 Web 光标代码。

### XR

| 结构体 | 用途 |
|--------|------|
| `FromWasmXrStartPresenting` | 开始 XR 呈现 |
| `FromWasmXrStopPresenting` | 停止 XR 呈现 |

### MIDI

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `FromWasmQueryMidiPorts` | — | 查询 MIDI 端口列表 |
| `FromWasmUseMidiInputs` | `input_uids: Vec<String>` | 选择使用的 MIDI 输入端口 |
| `FromWasmSendMidiOutput` | `uid`, `data: u32` | 发送 MIDI 输出数据 |

### 音频

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `FromWasmQueryAudioDevices` | — | 查询音频设备列表 |
| `FromWasmStartAudioOutput` | `web_device_id`, `context_ptr` | 启动音频输出 |
| `FromWasmStopAudioOutput` | — | 停止音频输出 |

### 视频播放

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `FromWasmPrepareVideoPlayback` | `video_id_lo/hi`, `texture_id`, `source_url`, `autoplay`, `should_loop` | 准备视频播放 |
| `FromWasmBeginVideoPlayback` | `video_id_lo/hi` | 开始播放 |
| `FromWasmPauseVideoPlayback` | `video_id_lo/hi` | 暂停播放 |
| `FromWasmResumeVideoPlayback` | `video_id_lo/hi` | 恢复播放 |
| `FromWasmMuteVideoPlayback` | `video_id_lo/hi` | 静音 |
| `FromWasmUnmuteVideoPlayback` | `video_id_lo/hi` | 取消静音 |
| `FromWasmSeekVideoPlayback` | `video_id_lo/hi`, `position_ms_lo/hi` | 跳转到指定时间 |
| `FromWasmCleanupVideoPlaybackResources` | `video_id_lo/hi` | 清理视频资源 |

### 线程（条件编译）

`FromWasmCreateThread` （仅在 `target_feature = "atomics"` 时编译）：包含 `context_ptr: u32` 和 `timer: u32`。

## 设计与约定

- 所有结构体派生 `#[derive(FromWasm)]`，该宏自动生成序列化代码
- 消息 ID 使用组合方式（`video_id_lo/hi`、`request_id_lo/hi`）编码 64 位标识符
- `WasmDataU8`、`WasmPtrF32`、`WasmPtrU32`、`WasmPtrU8` 用于传递二进制数据的内存指针
- 已注释的 `FromWasmWebSocketOpen/SendBinary/SendString` 表明 WebSocket 已被迁移到替代实现
