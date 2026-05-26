# `modal.rs` — 模态对话框

## 作用
实现一个全屏叠加的模态弹窗，带有半透明背景遮罩，支持点击外部关闭、Escape 键关闭、返回手势关闭。底层基于 `View` + `DrawList2d` 叠加层。

## 关键结构

### `ModalAction`
| 变体 | 说明 |
|------|------|
| `Dismissed` | 模态框被关闭 |
| `None` | 无操作 |

### `Modal`
| 字段 | 类型 | 说明 |
|------|------|------|
| `view` | `View`（`#[deref]`） | 内部 View |
| `draw_list` | `Option<DrawList2d>` | 叠加层绘制列表 |
| `draw_bg` | `DrawQuad` | 背景绘制 |
| `is_open` | `bool` | 是否打开 |
| `can_dismiss` | `bool`（默认 true） | 是否允许通过外部交互关闭 |

## 方法详解

### `on_after_new` / `on_after_apply`（ScriptHook）
- 新建时创建 `DrawList2d`；属性应用后强制重绘该列表，确保叠加层初始化

### `handle_event`（Widget）
- 仅在 `is_open == true` 时处理事件。先向内部 `content` 控件转发事件
- 主动消费背景区域的点击事件（`bg_area_hit`），防止穿透到模态下方的视图
- 当 `can_dismiss == true` 时，检查以下关闭条件：
  - 返回手势（`event.back_pressed()`）
  - 在背景上点击（FingerUp 不在 content 区域内）
  - 当 content 或 bg_view 有键盘焦点时按下 Escape 键
- 条件满足时生成 `ModalAction::Dismissed` action 并调用 `close`

### `draw_walk`（Widget）
- 使用 `draw_list.begin_overlay_reuse` 叠加绘制。先绘制背景，再在 `is_open` 时绘制遮罩层（`bg_view`）和内容层（`content`）
- 绘制完成后，调用 `cx.block_scrolling_except_within(content.area())` 限制滚动范围仅限于模态内容区域

### `open`
- 设置 `is_open = true`，强制重绘 `draw_list` 和 `draw_bg`
- 调用 `cx.set_key_focus(content.area())` 将键盘焦点设置到内容区域

### `close`
- 先检查 `is_open == false` 时直接返回（防止在已关闭时误调用导致焦点被夺走）
- 向内容控件发送 `ModalAction::Dismissed` action
- 设置 `is_open = false`，重绘 `draw_list` 和 `draw_bg`
- 调用 `cx.revert_key_focus()` 恢复焦点，`cx.unblock_scrolling()` 恢复滚动

### `dismissed`
- 检查 actions 中是否包含本模态的 `ModalAction::Dismissed` 事件

### `ModalRef` 方法
- `is_open` / `open` / `close` / `dismissed`：通过 `borrow`/`borrow_mut` 安全访问内部状态
