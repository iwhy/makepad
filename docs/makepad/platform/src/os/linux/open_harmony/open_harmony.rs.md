# open_harmony.rs

One-liner (EN): Core OpenHarmony platform implementation — EGL context/window management, event loop, touch/keyboard/IME handling, and platform operations.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/open_harmony/open_harmony.rs` (727 行)
- **核心作用**: OpenHarmony 平台的核心实现文件。管理 EGL 渲染上下文和窗口表面，实现主事件循环（接收来自 ArkTS 层的消息），处理触摸事件、键盘事件、IME 输入和平台操作（窗口创建、计时器、退出等）。

## 类型/结构体

### `CxOhosDisplay` — 显示封装

| 字段 | 类型 | 说明 |
|------|------|------|
| `libegl` | `LibEgl` | EGL 动态加载库 |
| `libgl` | `LibGl` | OpenGL ES 动态加载库 |
| `egl_display` | `EGLDisplay` | EGL 显示连接 |
| `egl_config` | `EGLConfig` | EGL 帧缓冲配置 |
| `egl_context` | `EGLContext` | EGL 渲染上下文 |
| `surface` | `EGLSurface` | EGL 窗口表面 |
| `window` | `*mut c_void` | 原生窗口句柄 |

方法: `destroy_surface`, `update_surface`, `swap_buffers`, `make_current` — 均为 `unsafe`。

### `CxOs` — 操作系统状态

| 字段 | 类型 | 说明 |
|------|------|------|
| `first_after_resize` | `bool` | 调整大小后首次重绘标记 |
| `display_size` | `Vec2d` | 显示尺寸（物理像素） |
| `dpi_factor` | `f64` | DPI 缩放因子 |
| `media` | `CxOpenHarmonyMedia` | 媒体子系统状态 |
| `quit` | `bool` | 退出主循环标记 |
| `timers` | `PollTimers` | 计时器管理器 |
| `raw_file` | `Option<RawFileMgr>` | 原生资源文件管理器 |
| `arkts_obj` | `Option<ArkTsObjRef>` | ArkTS 全局对象引用 |
| `start_time` | `Instant` | 应用启动时间戳 |
| `display` | `Option<CxOhosDisplay>` | 显示封装 |

## napi 导出函数

| 函数 | napi 名称 | 说明 |
|------|-----------|------|
| `ohos_ability_on_create(env, ark_ts)` | `onCreate` | 从 ArkTS 层接收初始化参数，提取设备信息、路径和资源管理器，发送 `FromOhosMessage::Init` 到 Rust 主循环 |

## 关键方法

### 初始化与启动

| 方法 | 说明 |
|------|------|
| `ohos_init(exports, env, startup)` | OpenHarmony 入口点。注册 XComponent 回调，一次性启动 `ohos_startup` 线程 |
| `ohos_startup(startup)` | 在独立线程中运行：初始化 mpsc 通道 → 运行启动闭包 → `wait_init` → 加载依赖 → `wait_surface_created` → 创建 EGL 上下文/表面 → `register_vsync_callback` → `main_loop` |
| `wait_init(from_ohos_rx)` | 等待接收 `Init` 消息，配置 `dpi_factor`、`os_type`、资源管理器和 ArkTS 引用 |
| `wait_surface_created(from_ohos_rx)` | 等待接收 `SurfaceCreated` 消息，返回窗口句柄 |
| `ohos_load_dependencies()` | 通过 `RawFileMgr` 读取应用 bundle 中的依赖文件（脚本、资源等）|

### 主事件循环

| 方法 | 说明 |
|------|------|
| `main_loop(from_ohos_rx)` | 主循环：阻塞直到收到 `VSync` 信号 → `handle_all_pending_messages` → `handle_other_events` → `handle_drawing` |
| `handle_all_pending_messages(rx)` | 非阻塞地消费所有待处理消息（在一帧内可能堆积的触摸/IME/表面事件）|
| `handle_other_events()` | 处理计时器、信号（Signal/Action）、视频更新、实时编辑、平台操作 |
| `handle_drawing()` | 检查脏 pass 和重绘需求，触发 `call_draw_event`、编译着色器、执行 `handle_repaint` |
| `handle_message(msg)` | 消息分发中心：处理 SurfaceCreated/Destroyed/Changed、触摸、文本输入、DeleteLeft、ResizeTextIME |

### 渲染

| 方法 | 说明 |
|------|------|
| `draw_pass_to_fullscreen(draw_pass_id)` | 将指定 pass 渲染到全屏：设置视口、清除颜色/深度、渲染视图列表、swap buffers |
| `handle_repaint()` | 计算需要重绘的 pass 列表，对全屏 pass 调用 `draw_pass_to_fullscreen`，对纹理 pass 调用 `draw_pass_to_texture` |

### 平台操作

| `CxOsOp` 处理 | 行为 |
|---------------|------|
| `CreateWindow` | 设置窗口几何属性，标记已创建 |
| `CreatePopupWindow` | 创建弹出窗口，设置位置/大小/父窗口 |
| `StartTimer / StopTimer` | 管理 `PollTimer` 插入/移除 |
| `Quit` | 设置 `quit = true` 退出主循环 |
| `ShowTextIME` | 调用 ArkTS 对象的 `showKeyBoard` 函数 |
| `HideTextIME` | 调用 ArkTS 对象的 `hideKeyBoard` 函数 |

### CxOsApi trait 实现

| 方法 | 行为 |
|------|------|
| `init_cx_os()` | 设置 `package_root = "makepad"`，调用 `native_load_dependencies` |
| `spawn_thread(f)` | 通过 `std::thread::spawn` 生成线程 |
| `open_url(url, in_place)` | 日志记录"未实现" |
| `seconds_since_app_start()` | 返回从 `start_time` 开始的秒数 |

## 实现细节

### 事件循环架构

```
ArkTS (UI线程)          Rust (渲染线程)
  │                        │
  │── onCreate() ──────►   │  wait_init()
  │                        │  ohos_load_dependencies()
  │── SurfaceCreated ──►   │  wait_surface_created()
  │                        │  创建 EGL context + surface
  │                        │  register_vsync_callback()
  │                        │
  │── VSync ────────────►  │  [main_loop]
  │── Touch ────────────►  │    ├ handle_all_pending_messages()
  │── TextInput ────────►  │    │  └ handle_message()
  │── DeleteLeft ────────► │    ├ handle_other_events()
  │── KeyBoard ──────────►  │    │  └ timers / signals / live_edit
  │                        │    └ handle_drawing()
  │                        │       └ draw_pass_to_fullscreen()
```

- Rust 渲染运行在独立线程上，通过 `mpsc::Receiver` 阻塞接收消息
- VSync 驱动渲染循环：每收到一个 VSync 信号，消费所有待处理消息，处理非渲染事件，然后执行绘制
- 触摸坐标除以 `dpi_factor` 从物理像素转到逻辑像素
- 表面变化事件（SurfaceChanged）触发窗口几何体更新和全屏重绘
- IME 键盘的显示/隐藏通过 `ArkTsObjRef::call_js_function` 调用 ArkTS 侧的方法
- `CxOhosDisplay::update_surface` 在 EGL 表面变化时销毁旧表面并创建新表面
