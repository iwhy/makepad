# `headless/event_loop.rs` — 无头事件循环

## 概述

本文件实现 Makepad 无头模式的事件循环，提供多种运行策略来适配不同场景：

1. **单帧模式** (`headless_single_frame`): 启动→渲染一帧→退出，适合 CI 测试和截图
2. **有限循环模式** (`headless_bounded_loop`): 渲染指定帧数后退出，适合基准测试
3. **标准输入协议模式** (`stdin_event_loop`): 通过 stdin/stdout 与 Studio 或 Flutter 宿主通信，适合交互式集成

---

## `HeadlessWindowState` — 窗口状态管理

```rust
struct HeadlessWindowState {
    created: bool,                    // 窗口是否已创建
    width: u32,                       // 窗口像素宽度
    height: u32,                      // 窗口像素高度
    dpi_factor: f64,                  // DPI 缩放因子
    frame_id: u64,                    // 帧序号（输出文件命名用）
    presentable_id: Option<PresentableImageId>, // Flutter 端可呈现图像 ID
}
```

`ensure_size_defaults()` 确保最小尺寸为 1280×720，DPI 因子为 1.0。

---

## 事件循环入口 — `Cx::event_loop()`

根据环境选择事件循环模式：
1. 如果 `should_run_stdin_loop_from_env()` 返回 true，进入**标准输入协议模式**
2. 如果设置了 `draw_cycles`，进入**有限循环模式**
3. 否则进入**单帧模式**

---

## 单帧模式 — `headless_single_frame()`

1. 发送 `Event::Startup`
2. 处理平台操作（创建默认窗口 1280×720）
3. 如果没有窗口创建，自动推送一个默认窗口
4. 触发 `Event::NextFrame`（如果有）
5. 如果需要绘制，调用 `headless_process_draw_cycle`

此模式在渲染完成后立即退出，适合一次性截图。

---

## 有限循环模式 — `headless_bounded_loop(draw_cycles)`

在主循环中执行 `draw_cycles` 次绘制：

```
while running && completed_cycles < draw_cycles:
    1. 检查 UI 信号（终止/脚本信号）
    2. 检查 Action 信号
    3. 分发网络运行时事件
    4. 分派定时器事件
    5. 处理平台操作
    6. 触发 NextFrame 事件
    7. headless_process_draw_cycle → 编译着色器 → 渲染所有通道
    8. completed_cycles += 1
    9. 睡眠 1ms（避免 CPU 空转）
```

---

## 标准输入协议模式 — `stdin_event_loop()`

这是 Flutter 集成和 Makepad Studio 远程控制的关键模式。

### 启动阶段

```
1. 设置 stdout 为 Studio 模式
2. 启动 stdin 读取线程 → 反序列化 StudioToApp JSON 消息 → 发送到 mpsc channel
3. write_stdout(AppToStudio::BeforeStartup)
4. call_event_handler(Event::Startup)
5. 处理平台操作
6. write_stdout(AppToStudio::AfterStartup)
```

### 主循环（while running）

从 mpsc channel 接收 `StudioToApp` 消息，分派事件：

| 消息 | 处理方式 |
|------|----------|
| `KeyDown` / `KeyUp` | 直接调用事件处理器 |
| `TextInput` | 文本输入事件 |
| `TextCopy` / `TextCut` | 剪贴板事件，返回 `AppToStudio::SetClipboard` |
| `MouseDown` | 处理 tap count，计算窗口内坐标，发送 `Event::MouseDown` |
| `MouseMove` | 跟踪鼠标按钮所在窗口，发送 `Event::MouseMove`，更新 hover/capture |
| `MouseUp` | 发送 `Event::MouseUp`，释放按钮和 hover |
| `Scroll` | 计算窗口内坐标，发送 `Event::Scroll` |
| `TweakRay` | 发送 `Event::TweakRay`（Studio 调试工具）|
| `WindowGeomChange` | 更新窗口几何信息和 DPI，触发 `redraw_all` |
| `Swapchain` | 更新窗口尺寸和 presentable_id，触发 `redraw_all` |
| `Tick` | 检查信号/定时器 → 处理平台操作 → 渲染 → 请求下一帧 |
| `RunViewFrameRequest` | 空操作 |
| 其他 | 调用 `dispatch_studio_msg` 分发 |

### 渲染响应

在 `Tick` 和其他消息的处理中，`headless_process_draw_cycle` 被调用：

1. 如果 no_draw 模式：发送 `Event::Draw`，标记初始化
2. 否则：发送 `Event::Draw`，调用 `headless_compile_shaders()`
3. 调用 `headless_emit_frames(windows, send_protocol=true)`

如果检测到需要继续渲染（有待处理的定时器或 next_frame），发送 `AppToStudio::RequestAnimationFrame` 请求宿主发起下一轮 `Tick`。

每个渲染周期后发送 `AppToStudio::DrawCompleteAndFlip` 通知宿主交换链可用。

---

## `headless_process_draw_cycle()` — 绘制周期处理

```rust
fn headless_process_draw_cycle(
    &mut self,
    windows: &mut Vec<HeadlessWindowState>,
    send_protocol: bool,  // 是否通过协议输出
    time_now: f64,
) -> bool
```

返回 `true` 表示渲染了帧，`false` 表示无输出。

执行步骤：
1. `call_draw_event(time_now)` — 触发 Draw 事件
2. `headless_compile_shaders()` — 编译待编译着色器
3. 如果是协议模式且有截图请求：`headless_render_all_passes(time_now)`
4. 否则：`headless_emit_frames(windows, send_protocol, time_now)` → 渲染+编码+输出

---

## `headless_emit_frames()` — 帧输出

将 `Framebuffer` 转换为可传输/可保存的格式：

1. 调用 `headless_render_all_passes(time_now)` 获取所有 `(window_id, Framebuffer)`
2. 对每个帧缓冲：
   - 调用 `fb.to_rgba8()` — 反预乘 + 量化到 u8
   - 调用 `encode_png_rgba(width, height, &rgba)` — 编码为 PNG
   - **协议模式**: 通过 `write_stdout_msg` 发送 `AppToStudio::Screenshot`（含截图请求 ID）
   - **文件模式**: 写入 `${output_dir}/window_{id}_frame_{frame_id:06}.png`
3. 协议模式下发送 `AppToStudio::DrawCompleteAndFlip`

帧输出目录通过 `MAKEPAD_HEADLESS_OUT_DIR` 环境变量或 `CxOs.frame_dir` 设置。

---

## `headless_handle_platform_ops()` — 平台操作处理

从 `self.platform_ops` 队列中弹出并处理操作：

| 操作 | 处理 |
|------|------|
| `CreateWindow` | 创建窗口状态，设置几何信息，发送 `CreateWindow` 协议消息 |
| `CreatePopupWindow` | 创建弹出窗口，设置父窗口和键盘捕获 |
| `ResizeWindow` | 更新窗口尺寸 |
| `SetCursor` | 协议模式发送 `SetCursor` 消息 |
| `StartTimer` | 添加定时器到 `PollTimers` |
| `StopTimer` | 移除定时器 |
| `CopyToClipboard` | 协议模式发送 `SetClipboard` 消息 |
| `Quit` | 返回 `false` 停止事件循环 |

---

## `CxOsApi` 实现

| 方法 | 说明 |
|------|------|
| `init_cx_os()` | 初始化 `start_time`，读取 `--no-draw` 和 `--draw-cycles` 参数，加载包根路径 |
| `spawn_thread(f)` | 使用 `std::thread::spawn` 生成线程 |
| `seconds_since_app_start()` | 当前时间与 `start_time` 的差值（秒）|
| `open_url()` | 空操作（在 headless 模式下忽略） |

---

## `write_stdout_msg()` — 协议消息输出

将 `AppToStudio` 消息序列化为 JSON 行并通过 stdout 发送：
```rust
fn write_stdout_msg(msg: &AppToStudio) {
    let _ = io::stdout().write_all(msg.to_json().as_bytes());
    let _ = io::stdout().write_all(b"\n");
    let _ = io::stdout().flush();
}
```

这是 headless 模式与外部进程（Flutter/Makepad Studio）的标准通信接口，使用 JSON Lines 协议。
