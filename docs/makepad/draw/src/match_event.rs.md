# `draw/src/match_event.rs` — MatchEvent：统一事件分发 Trait

## 设计目标

`MatchEvent` 是 Makepad 框架中**所有 UI 组件的事件入口 trait**。任何需要响应事件的类型（Window、View、Button、App 等）都应实现此 trait。

核心思想是**双路径事件分发**：
1. `match_event` — 完整的事件匹配，覆盖所有事件类型
2. `match_event_with_draw_2d` — 简化版的帧循环路径，只为 `Draw` 事件创建 2D 绘制上下文

## Trait 定义

```rust
pub trait MatchEvent {
    fn handle_startup(&mut self, _cx: &mut Cx) {}
    fn handle_shutdown(&mut self, _cx: &mut Cx) {}
    // ... 其他 handler
    fn match_event(&mut self, cx: &mut Cx, event: &Event) { ... }
    fn match_event_with_draw_2d(&mut self, cx: &mut Cx, event: &Event) -> Result<(), ()> { ... }
}
```

所有 handler 方法都有**默认空实现**，实现者只需覆写需要的 handler。

---

## Handler 方法列表

### 应用生命周期

| Handler | 触发事件 | 用途 |
|---------|----------|------|
| `handle_startup(cx)` | `Event::Startup` | 应用启动时初始化 |
| `handle_shutdown(cx)` | `Event::Shutdown` | 应用退出时清理 |
| `handle_quit_requested(cx, e) -> bool` | `Event::QuitRequested` | 系统请求退出窗口（返回 `true` 表示已处理，阻止默认行为） |

`handle_quit_requested` 通过 `e.handled` 的 `Cell<bool>` 标记来防止事件重复处理。

### 前台/后台/暂停/恢复

| Handler | 触发事件 | 用途 |
|---------|----------|------|
| `handle_foreground(cx)` | `Event::Foreground` | 应用回到前台（移动端/桌面上下文恢复） |
| `handle_background(cx)` | `Event::Background` | 应用进入后台 |
| `handle_pause(cx)` | `Event::Pause` | 应用被系统暂停 |
| `handle_resume(cx)` | `Event::Resume` | 应用从暂停恢复 |

### 窗口焦点

| Handler | 触发事件 | 用途 |
|---------|----------|------|
| `handle_window_got_focus(cx, window_id)` | `Event::WindowGotFocus` | 窗口获得焦点 |
| `handle_window_lost_focus(cx, window_id)` | `Event::WindowLostFocus` | 窗口失去焦点 |

### 帧循环与重绘

| Handler | 触发事件 | 用途 |
|---------|----------|------|
| `handle_next_frame(cx, e)` | `Event::NextFrame` | 下一帧回调（用于 requestAnimationFrame 式场景） |
| `handle_draw(cx, e)` | `Event::Draw` | 绘制事件（平台级，在 CxDraw/Cx2d 创建前） |
| `handle_draw_2d(cx: &mut Cx2d)` | `Event::Draw`（2D 路径） | 2D 绘制入口（通过 `match_event_with_draw_2d`） |

### 输入事件

| Handler | 触发事件 | 用途 |
|---------|----------|------|
| `handle_key_down(cx, e)` | `Event::KeyDown` | 按键按下 |
| `handle_key_up(cx, e)` | `Event::KeyUp` | 按键释放 |
| `handle_back_pressed(cx) -> bool` | `Event::BackPressed` | 安卓返回键（返回 `true`=已处理，不再传播） |

`handle_back_pressed` 使用了与 `handle_quit_requested` 相同的 `handled` Cell 防重复机制。

### 动作与信号

| Handler | 触发事件 | 用途 |
|---------|----------|------|
| `handle_action(cx, e)` | 包含在 `Actions` 中 | 处理单个 Action |
| `handle_actions(cx, actions)` | `Event::Actions` | 批量处理 Actions（默认实现：遍历调用 `handle_action`） |
| `handle_signal(cx)` | `Event::Signal` | OS 信号（如 SIGINT） |

### 多媒体设备

| Handler | 触发事件 | 用途 |
|---------|----------|------|
| `handle_audio_devices(cx, e)` | `Event::AudioDevices` | 音频设备列表变化 |
| `handle_midi_ports(cx, e)` | `Event::MidiPorts` | MIDI 端口变化 |
| `handle_video_inputs(cx, e)` | `Event::VideoInputs` | 视频输入设备变化 |

### HTTP 网络请求

| Handler | 触发事件 | 用途 |
|---------|----------|------|
| `handle_http_response(cx, request_id, response)` | 在 `NetworkResponses` 中 | HTTP 完整响应到达 |
| `handle_http_request_error(cx, request_id, err)` | 在 `NetworkResponses` 中 | HTTP 请求出错 |
| `handle_http_progress(cx, request_id, progress)` | 在 `NetworkResponses` 中 | 下载进度通知 |
| `handle_http_stream(cx, request_id, data)` | 在 `NetworkResponses` 中 | HTTP 流式数据块 |
| `handle_http_stream_complete(cx, request_id, data)` | 在 `NetworkResponses` 中 | HTTP 流式传输完成 |
| `handle_network_responses(cx, e)` | `Event::NetworkResponses` | 批量分发网络响应 |

`handle_network_responses` 的默认实现遍历 `NetworkResponsesEvent` 并分发到对应的 HTTP handler。注意 `WsOpened`、`WsMessage`、`WsClosed`、`WsError` 等 WebSocket 响应当前没有对应 handler，直接忽略。

### 计时器

| Handler | 触发事件 | 用途 |
|---------|----------|------|
| `handle_timer(cx, e)` | `Event::Timer` | 定时器超时 |

---

## `match_event(cx, event)` — 主事件匹配器

```rust
fn match_event(&mut self, cx: &mut Cx, event: &Event) {
    match event {
        Event::Startup => self.handle_startup(cx),
        Event::Shutdown => self.handle_shutdown(cx),
        Event::QuitRequested(e) if !e.handled.get() => {
            let was_handled = self.handle_quit_requested(cx, e);
            e.handled.set(was_handled);
        }
        Event::Foreground => self.handle_foreground(cx),
        Event::Background => self.handle_background(cx),
        Event::Pause => self.handle_pause(cx),
        Event::Resume => self.handle_resume(cx),
        Event::Signal => self.handle_signal(cx),
        Event::WindowGotFocus(window_id) => self.handle_window_got_focus(cx, window_id),
        Event::Timer(te) => self.handle_timer(cx, te),
        Event::WindowLostFocus(window_id) => self.handle_window_lost_focus(cx, window_id),
        Event::NextFrame(e) => self.handle_next_frame(cx, e),
        Event::Actions(e) => self.handle_actions(cx, e),
        Event::Draw(e) => self.handle_draw(cx, e),
        Event::AudioDevices(e) => self.handle_audio_devices(cx, e),
        Event::MidiPorts(e) => self.handle_midi_ports(cx, e),
        Event::VideoInputs(e) => self.handle_video_inputs(cx, e),
        Event::NetworkResponses(e) => self.handle_network_responses(cx, e),
        Event::KeyDown(e) => self.handle_key_down(cx, e),
        Event::KeyUp(e) => self.handle_key_up(cx, e),
        Event::BackPressed { handled } if !handled.get() => {
            let was_handled = self.handle_back_pressed(cx);
            handled.set(was_handled);
        }
        _ => (),
    }
}
```

### 实现说明

1. **详尽匹配**：覆盖了所有 `Event` 变体
2. **handler Cell 保护**：`QuitRequested` 和 `BackPressed` 使用条件匹配 `if !handled.get()`，防止同一事件被处理两次
3. **兜底**：`_ => ()` 处理未被匹配的事件变体（如 `Mouse`、`Touch`、`Scroll` 等底层输入事件）

---

## `match_event_with_draw_2d(cx, event)` — 带 2D 上下文的匹配

```rust
fn match_event_with_draw_2d(&mut self, cx: &mut Cx, event: &Event) -> Result<(), ()> {
    match event {
        Event::Draw(e) => {
            let mut cx_draw = CxDraw::new(cx, e);
            let mut cx_2d = Cx2d::new(&mut cx_draw);
            self.handle_draw_2d(&mut cx_2d);
            Ok(())
        }
        e => {
            self.match_event(cx, e);
            Err(())
        }
    }
}
```

### 实现说明

1. **如果事件是 `Draw`**：
   - 创建 `CxDraw`（绑定 Cx + DrawEvent）
   - 创建 `Cx2d`（包装 CxDraw，添加 Turtle 布局支持）
   - 调用 `handle_draw_2d(&mut cx_2d)`——进入 2D 绘制路径
   - 返回 `Ok(())` 表示已处理（Draw 事件由此 route 消费）
2. **如果是其他事件**：
   - 委托给 `match_event` 处理
   - 返回 `Err(())` 表示未通过此路径处理（不影响事件的进一步传播）

### 设计意图

`match_event_with_draw_2d` 是**应用主循环推荐使用的入口**。它将 `Draw` 事件自动引导到 2D 绘制路径（自动创建 CxDraw → Cx2d 链），而将其他事件转发到通用 `match_event` 分发器。
