# widget_match_event.rs — WidgetMatchEvent Trait 与事件分发

## 文件概述

定义了 `WidgetMatchEvent` trait，为脚本化 Widget 提供基于 `match_event!` 宏的事件分发能力。所有派生 `Script` 和 `ScriptHook` 的结构体都自动实现该 trait。

---

## `WidgetMatchEvent` trait

```rust
pub trait WidgetMatchEvent {
    fn match_event(&mut self, cx: &mut Cx, event: &Event);
}
```

`match_event` 是 Makepad 事件系统的入口方法，由 `match_event!` 宏生成具体实现。其职责是按照优先级顺序检查事件类型并调用对应的 handler：

### 事件处理优先级

1. **App 级事件**：
   - `AppEvent` — 应用生命周期事件（启动、退出、暂停恢复）
   - `SignalEvent` — 平台信号（SIGINT、SIGTERM 等）

2. **绘制/帧事件**：
   - `DrawEvent` — 绘图请求，在需要重绘时触发
   - `FrameEvent` — 逐帧回调，驱动动画循环
   - `ResizeEvent` — 窗口/视图大小变化

3. **输入事件**：
   - `MouseDownEvent` / `MouseUpEvent` — 鼠标按下/释放
   - `MouseMoveEvent` — 鼠标移动（包含拖拽）
   - `MouseScrollEvent` — 滚轮滚动
   - `KeyDownEvent` / `KeyUpEvent` — 键盘按下/释放
   - `TextInputEvent` / `TextCopyEvent` / `TextCutEvent` — 文本输入/剪贴板

4. **Widget 系统事件**：
   - `WidgetEvent` — 子 Widget 冒泡上来的自定义 action
   - `WidgetPtrEvent` — 跨 Widget 引用的直接事件发送

### 各方法的默认实现

`WidgetMatchEvent` trait 中的 `match_event` 默认调用顺序（由 `match_event!` 宏生成）：

```rust
fn match_event(&mut self, cx: &mut Cx, event: &Event) {
    match event {
        Event::App(e) => self.match_app_event(cx, e),
        Event::Draw(e) => self.match_draw_event(cx, e),
        Event::Frame(e) => self.match_frame_event(cx, e),
        Event::Resize(e) => self.match_resize_event(cx, e),
        Event::MouseDown(e) => self.match_mouse_down(cx, e),
        Event::MouseUp(e) => self.match_mouse_up(cx, e),
        // ... 继续所有事件类型
    }
}
```

### `match_event` 中的 Action 冒泡

当 Widget 的 `handle_event` 方法没有消费事件时，事件冒泡到父 Widget 的 `match_event`。核心路由逻辑：

```rust
fn handle_event(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope) {
    self.match_event(cx, event);
    // 如果事件是 WidgetEvent，则触发 handle_actions
    if let Event::WidgetEvent(e) = event {
        if let Some(actions) = e.get_action_set_for(self.ui.widget_ptr().widget_uid()) {
            self.handle_actions(cx, actions);
        }
    }
}
```

---

## HandleActions trait

```rust
pub trait HandleActions {
    fn handle_actions(&mut self, cx: &mut Cx, actions: &Actions);
}
```

当子 Widget 产生 Action（按钮点击、文本变更等）时，通过 WidgetEvent 路由到父组件的 `handle_actions` 方法。`HandleActions` trait 默认空实现，用户可覆盖。

---

## ActionId 与 actions! 宏

Action 通过 `actions!` 宏生成 ID 映射，在 `handle_actions` 中匹配：

```rust
fn handle_actions(&mut self, cx: &mut Cx, actions: &Actions) {
    if self.ui.button(id!(my_btn)).clicked(actions) {
        log!("Button clicked!");
    }
}
```

- `actions!` 宏：将 Widget ID 转换为 `ActionId` 集合，用于在冒泡事件中快速查找。
- `id!()` 宏：编译期生成 `LiveId`，用于标识 Widget 实例。
- `ids!()` 宏：生成多个 `LiveId` 的集合。

---

## 可覆盖的 Handler 方法

`match_event!` 宏为每种事件类型生成对应的 handler：

| Handler | 事件类型 | 典型用途 |
|---------|----------|----------|
| `match_app_event` | AppEvent | 应用启动/退出初始化 |
| `match_draw_event` | DrawEvent | 强制重绘 |
| `match_frame_event` | FrameEvent | 动画循环更新 |
| `match_resize_event` | ResizeEvent | Responsive 布局适配 |
| `match_mouse_down`/`up` | Mouse 按钮事件 | 点击/释放处理 |
| `match_mouse_move` | MouseMove | 悬停/拖拽 |
| `match_mouse_scroll` | MouseScroll | 滚动处理 |
| `match_key_down`/`up` | Key 事件 | 键盘导航/快捷键 |
| `match_text_input`/`copy`/`cut` | 文本事件 | 输入框处理 |

所有 handler 默认空实现，用户根据需要在派生结构体的 `impl` 块中覆盖。
