# windowing_backend.rs — 窗口系统后端选择与 CxOs 定义

**文件路径**: `platform/src/os/linux/windowing_backend.rs` (185 行)
**核心功能**: 检测 Linux 桌面环境中的窗口协议（X11/Wayland），启动对应的事件循环，定义 `CxOs` 结构体。

## 协议检测

### `detect_windowing_protocol() -> WindowingProtocol`
按优先级检测当前窗口协议：
1. `--linux-backend=x11` 或 `--linux-backend=wayland` 命令行参数
2. `WAYLAND_DISPLAY` 环境变量 → Wayland
3. `DISPLAY` 环境变量 → X11
4. 默认回退 → X11

### `WindowingProtocol` 枚举
```rust
pub enum WindowingProtocol { X11, Wayland }
```

## 事件循环

### `Cx::event_loop(cx)`
Main 事件循环入口：
1. 检测窗口协议
2. 打印环境信息（`WAYLAND_DISPLAY`、`DISPLAY`、`XDG_SESSION_TYPE`、`XDG_CURRENT_DESKTOP`）
3. 根据检测结果启动对应后端：
   - Wayland → `wayland_event_loop` → `super::wayland::linux_wayland::wayland_event_loop`
   - X11 → `x11_event_loop` → `super::x11::linux_x11::x11_event_loop`

### `Cx::handle_networking_events()`
处理网络运行时事件。

## `CxOsApi` 实现

- `init_cx_os` — 记录启动时间、设置包根目录、加载原生依赖
- `spawn_thread` — 使用 `std::thread::spawn`
- `seconds_since_app_start` — 返回自启动以来的秒数
- `open_url` — 暂未实现

## `CxOs` 结构体

Linux 平台的操作系统状态：
```rust
pub struct CxOs {
    pub media: CxLinuxMedia,          // 音视频媒体子系统
    pub stdin_timers: PollTimers,     // stdin 轮询定时器
    pub start_time: Option<Instant>,  // 启动时间
    pub opengl_cx: Option<OpenglCx>,  // OpenGL 上下文
    pub video_players: HashMap<LiveId, LinuxVideoPlayer>,  // 视频播放器
    pub gstreamer: Option<LibGStreamer>,  // GStreamer 实例
}
```

### `CxOs::gl()`
返回当前 OpenGL 函数表引用（panic 如果没有 OpenGL 上下文）。
