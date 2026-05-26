# Linux Direct 平台实现

## 概述

`linux_direct.rs` 是 Linux Direct（DRM/GBM/EGL）平台的主实现文件。它定义了完整的事件循环、OpenGL 渲染流程、平台操作处理和窗口管理。

## 核心类型

### `DirectApp`
Direct 平台应用状态：
- `timers` — `SelectTimers` 定时器管理
- `drm` — DRM/KMS 显示状态
- `egl` — EGL 渲染上下文
- `raw_input` — 原始输入设备
- `dpi_factor` — DPI 缩放因子

### `CxOs`
平台相关状态：
- `media` — `CxLinuxMedia` 媒体子系统
- `start_time` — 应用启动时间

## 关键方法

### `Cx::event_loop(cx)`
主事件循环：
1. 设置 `OsType::LinuxDirect` 和 `GpuPerformance::Tier1`
2. 调用 `Startup` 事件和首次渲染
3. 创建 `DirectApp`
4. 启动 8ms 定时器（timer 0，用于轮询信号和媒体事件）
5. 循环：更新定时器 → 处理定时器事件 → 轮询原始输入 → 渲染（`DirectEvent::Paint`）
6. 循环在 `EventFlow::Exit` 时终止

### `direct_event_callback(direct_app, event) -> EventFlow`
事件分发中心：

| 事件类型 | 处理 |
|----------|------|
| `Paint` | 调用 `handle_repaint`，执行 OpenGL 渲染 |
| `MouseDown` | DPI 缩放后派发 `MouseDown` 事件 |
| `MouseMove` | DPI 缩放后派发 `MouseMove` 事件，更新悬停和捕获 |
| `MouseUp` | DPI 缩放后派发 `MouseUp` 事件 |
| `Scroll` | DPI 缩放后派发 `Scroll` 事件 |
| `KeyDown` | 通过 `keyboard.process_key_down` 处理按键，派发事件 |
| `KeyUp` | 通过 `keyboard.process_key_up` 处理，派发事件 |
| `TextInput` | 派发文本输入事件 |
| `Timer` | timer 0：处理 UI 信号、媒体信号、脚本信号、网络事件；其他：脚本定时器和事件 |

### `handle_platform_ops(direct_app) -> EventFlow`
平台操作队列处理：

| 操作 | 处理 |
|------|------|
| `CreateWindow` | 设置窗口几何（全屏、DPI） |
| `CreatePopupWindow` | 创建弹出窗口 |
| `StartTimer` / `StopTimer` | 定时器管理 |
| `HttpRequest` / `CancelHttpRequest` | HTTP 网络请求 |
| `Quit` | 返回 `EventFlow::Exit` |
| `ResizeWindow` / `RepositionWindow` | 窗口几何更新 |
| `CheckPermission` / `RequestPermission` | 始终返回 `Granted` |
| 其他 | 记录 `Not implemented` 错误 |

### `draw_pass_to_fullscreen(draw_pass_id, direct_app)`
OpenGL 渲染到全屏：
1. 设置渲染通道
2. `glViewport` 设置为全屏尺寸
3. 清理颜色/深度缓冲区
4. `render_view` 渲染实际内容
5. `drm.swap_buffers_and_wait` 翻页

### `handle_repaint(direct_app)`
处理所有脏渲染通道的重新绘制，排序后逐个渲染。

## CxOsApi 实现

| 方法 | 实现 |
|------|------|
| `init_cx_os` | 加载原生依赖，设置 `package_root` |
| `spawn_thread` | 通过 `std::thread::spawn` 创建系统线程 |
| `open_url` | 未实现 |
| `seconds_since_app_start` | 使用 `Instant::now() - start_time` |

## 实现说明

- 事件循环全程使用 DPI 缩放因子通过 `dpi_override_scale` 转换坐标。
- 所有权限检查自动返回 `Granted`（Linux Direct 平台无权限系统）。
- 使用 8ms 固定间隔定时器轮询信号（≈ 120Hz），匹配常见的显示刷新率。
- `handle_media_signals` 实际委托给 `CxLinuxMedia`（linux_media 模块），而非 Android 的 `CxAndroidMedia`。
- 应用退出时调用 `Event::Shutdown` 事件。
