# `popup_notification.rs` — 弹出通知

## 作用
提供一个全屏覆盖的弹出通知容器，通常用于右下角显示临时消息。底层基于 `View` + `DrawList2d` 叠加层。

## 关键结构

### `PopupNotification`
| 字段 | 类型 | 说明 |
|------|------|------|
| `source` | `ScriptObjectRef` | 脚本对象引用 |
| `view` | `View`（`#[deref]`） | 代理内部 View |
| `draw_list` | `Option<DrawList2d>` | 叠加绘制列表 |
| `draw_bg` | `DrawQuad` | 背景绘制 |
| `opened` | `bool` | 是否打开 |

## 方法详解

### `on_after_new`（ScriptHook）
- 在脚本新建时调用。创建一个 `DrawList2d` 实例，用于在叠加层上绘制通知内容

### `on_after_apply`（ScriptHook）
- 属性应用后触发。通过 `vm.with_cx_mut` 获取上下文，强制重绘 `draw_list`，确保初始状态可见

### `handle_event`（Widget）
- 仅在 `opened == true` 时才转发事件给内部 view；关闭时忽略所有事件，实现"不可见不响应"

### `draw_walk`（Widget）
- 同样仅在打开时绘制。使用 `draw_list.begin_overlay_reuse` 启动叠加层复用模式
- 用 `cx.current_pass_size` 获取当前 pass 尺寸，调用 `cx.begin_root_turtle` 启动根级海龟布局
- 在其中依次绘制 `draw_bg` 和内部 view 的所有子控件，最后结束 pass 和绘制列表

### `open`
- 设置 `opened = true`，强制重绘 `draw_list` 使首次打开立即可见（即使叠加视图尚未建立可复用的绘制区域）
- 调用 `self.redraw(cx)` 触发常规重绘

### `close`
- 设置 `opened = false`，同样重绘 `draw_list` 清除最后一帧残留
- 额外重绘 `draw_bg` 确保背景也更新

### `PopupNotificationRef` 方法
- `is_open` / `open` / `close`：通过 `borrow`/`borrow_mut` 安全地访问内部状态，提供对外 API
