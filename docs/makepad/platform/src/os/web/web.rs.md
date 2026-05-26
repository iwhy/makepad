# web.rs — Web 平台核心集成与 WASM 消息桥接

**文件路径**: `/home/ubuntu/_github/makepad/platform/src/os/web/web.rs` (1205 行)
**核心作用**: Web 平台的 `Cx` 上下文扩展，实现 WASM ↔ JavaScript 双向消息处理、`CxOsApi` trait、平台操作队列处理、线程管理及 WASM 导出入口。

## 关键数据结构

### `CxOs` — Web 平台操作系统状态

```rust
pub struct CxOs {
    pub(crate) window_geom: WindowGeom,      // 窗口几何信息
    pub from_wasm: Option<FromWasmMsg>,       // 待发送的 FromWasm 消息
    pub(crate) vertex_buffers: usize,         // 顶点缓冲区计数器
    pub(crate) index_buffers: usize,          // 索引缓冲区计数器
    pub(crate) vaos: usize,                   // VAO 计数器
    pub(crate) to_wasm_js: Vec<String>,       // JS ToWasm 类定义代码
    pub(crate) from_wasm_js: Vec<String>,     // JS FromWasm 类定义代码
    pub(crate) media: CxWebMedia,             // 音频/MIDI 媒体管理
}
```

## 核心方法

### URL 处理

| 方法 | 说明 |
|------|------|
| `normalize_web_pathname` | 规范化 Web 路径：空值→`/`，补全前导 `/` |
| `split_web_location` | 将 URL 拆分为 (pathname, search, hash) 三元组，跳过 scheme |
| `update_web_location_state` | 更新 `OsType::Web` 中的位置状态，有变化返回 true |

### `process_to_wasm` — WASM 主入口点（核心事件循环）

这是从 JavaScript 进入 Rust 代码流的唯一入口。接收从 JS 序列化的 `ToWasmMsg`，通过 `LiveId` 识别消息类型并分发处理：

| 消息 ID | 处理内容 |
|---------|---------|
| `ToWasmInit` | 初始化：CPU 核心数、GPU 信息、浏览器信息、窗口几何；触发 `Startup` 事件 |
| `ToWasmResizeWindow` | 窗口大小变化，DPI 因子转换，触发 `WindowGeomChange` |
| `ToWasmAnimationFrame` | 动画帧回调，调用 `call_next_frame_event` |
| `ToWasmTouchUpdate` | 触摸事件更新，DPI 缩放坐标，处理 `TouchUpdate` |
| `ToWasmMouseDown/Move/Up` | 鼠标事件，DPI 缩放，tap 计数与悬停区域管理 |
| `ToWasmScroll` | 滚动事件，DPI 缩放 |
| `ToWasmKeyDown/Up` | 键盘事件，调用 `process_key_down/up` |
| `ToWasmTextInput` | 文本输入事件 |
| `ToWasmTextCopy` | 剪贴板复制响应 |
| `ToWasmSignal` | 信号处理（媒体、脚本、Action 接收器） |
| `ToWasmAppLifecycle` | 应用生命周期（Foreground/Background/Pause/Resume/Shutdown） |
| `ToWasmTimerFired` | 定时器触发 |
| `ToWasmWindowGotFocus/LostFocus` | 窗口焦点变化 |
| `ToWasmRedrawAll` | 全局重绘请求 |
| `ToWasmPaintDirty` | 标记绘制脏状态 |
| `ToWasmLiveFileChange` | 热重载文件变化 |
| `ToWasmLocationChange` | 浏览器 URL 位置变化 |
| `ToWasmHTTPResponse` | HTTP 响应返回 |
| `ToWasmHttpRequestError` | HTTP 请求错误 |
| `ToWasmHttpResponseProgress` / `ToWasmHttpUploadProgress` | HTTP 上传/下载进度 |
| `ToWasmPermissionResult` | 权限查询结果 |
| `ToWasmVideoPlaybackPrepared` | 视频播放就绪 |
| `ToWasmVideoTextureUpdated` | 视频纹理更新 |
| `ToWasmVideoPlaybackCompleted` | 视频播放完成 |
| `ToWasmVideoPlaybackResourcesReleased` | 视频资源释放 |
| `ToWasmAudioDeviceList` | 音频设备列表 |
| `ToWasmMidiPortList` | MIDI 端口列表 |
| `ToWasmMidiInputData` | MIDI 输入数据 |
| 其他 | 通过 `Event::ToWasmMsg` 转发用户自定义消息 |

处理完成后执行：绘制事件、网络事件、热重载、平台操作队列、媒体信号，最后按需请求动画帧。

### `handle_repaint` — 重绘管理

```rust
pub fn handle_repaint(&mut self, time: f64)
```
计算绘制 Pass 的重绘顺序，对每个 Pass 根据其父类型（Window/DrawPass/None/Xr）调用相应渲染方法。

### `handle_platform_ops` — 平台操作队列处理

将 `CxOsOp` 队列中的操作转换为相应的 `FromWasm` 消息：

| 操作 | 说明 |
|------|------|
| `CreateWindow` | 设置文档标题，继承 DPI 因子，触发 WindowGeomChange |
| `CreatePopupWindow` | 创建弹出窗口（记录位置、大小、父窗口） |
| `FullscreenWindow` / `NormalizeWindow` | 全屏/恢复 |
| `ShowTextIME` / `HideTextIME` | IME 输入法控制 |
| `SetCursor` | 鼠标光标设置 |
| `StartTimer` / `StopTimer` | 定时器管理 |
| `HttpRequest` / `CancelHttpRequest` | HTTP 请求 |
| `CheckPermission` / `RequestPermission` | 权限检查与请求 |
| `PrepareVideoPlayback` / `Begin/Pause/Resume/Mute/Unmute/Seek/Cleanup` | 视频播放全生命周期 |
| `SetVideoVolume` / `SetVideoPlaybackRate` | 视频音量/速率（当前空操作） |
| `XrStartPresenting` / `XrStopPresenting` | XR 呈现 |
| `UpdateVideoSurfaceTexture` | 视频纹理更新（Web 端无操作） |

视频播放初始化拒绝 `InMemory`、`Filesystem`、`Camera`、`PlaybackSession` 等来源，仅支持 `Network(url)`。

### `CxOsApi` Trait 实现

| 方法 | 实现 |
|------|------|
| `init_cx_os` | 注册网络后端 Shim，生成 ToWasm/FromWasm 的 JS 类定义代码（含 WebGL、音频、MIDI、视频等所有消息类型），原子化支持线程创建 |
| `seconds_since_app_start` | 返回 0.0（Web 无此功能） |
| `spawn_thread` | 原子化：通过 `FromWasmCreateThread` 在 JS 侧创建线程；非原子化：空操作 |
| `open_url` | 发送 `FromWasmOpenUrl` 消息 |
| `browser_update_url` | 更新浏览器 URL 并同步状态 |
| `browser_history_go` | 浏览器历史导航（delta ≠ 0 时发送） |
| `default_window_size` | 返回保存的窗口内部大小 |

### WASM 导出函数

- `wasm_get_js_message_bridge` — 生成包含所有 ToWasm/FromWasm 类定义的 JS 对象字符串
- `wasm_check_signal` — 检查信号标志（1=UI 信号，2=Action 信号）
- `wasm_init_panic_hook` — 初始化 Rust panic 钩子
- `wasm_thread_entrypoint` — 线程入口（原子化）
- `wasm_thread_timer_entrypoint` — 定时器线程入口（原子化）
- `wasm_thread_alloc_tls_and_stack` — 线程 TLS/栈分配（原子化）
- `js_time_now` — 外部 JS 函数：获取当前时间

## 实现细节

- `process_to_wasm` 使用块跳过机制（`read_block_skip`/`block_skip`）支持批量消息高效处理
- 动画帧时间用于驱动 `need_redrawing`、`call_draw_event`、`webgl_compile_shaders`、`handle_repaint`
- 网络响应累积到 `Vec<NetworkResponse>` 后统一通过脚本网络事件和 `Event::NetworkResponses` 分发
- `BASE_ADDR` 静态变量用于 WASM 内存基地址

## 平台适配

`spawn_thread` 和 `spawn_timer_thread` 的条件编译提供两种实现：
- `target_feature = "atomics"`：使用 `FromWasmCreateThread` 在 JS 侧创建真正的 Web Worker 线程
- 无原子特性：空操作（单线程模式）
