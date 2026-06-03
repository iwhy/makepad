# `desktop_run_list.rs` — Studio 可运行列表组件

## 文件作用

在左侧面板中显示当前 mount 下所有可执行目标（runnable items）的列表，支持点击运行。

## script_mod 定义

### 子组件

**`RunPlayIcon`**: 三角形播放图标（SDF 绘制）：
```
sdf.move_to(3, 2) → sdf.line_to(11, 7) → sdf.line_to(3, 12) → close_path → fill
```
hover 时颜色变亮。

**`RunListItem`**: 列表行（View）：
- 34px 高度，交替背景色（`is_even`）
- 包含 `RunPlayIcon` 和 `row_button`（全宽按钮）
- `Animator` 控制 hover 效果（cursor: Hand, icon/text 颜色渐变）

**`RunListEmpty`**: 空状态占位项。

**`DesktopRunList`**: 完整列表组件：
```
list := PortalList {
    Item := RunListItem
    Empty := RunListEmpty
}
```

## 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct DesktopRunList {
    #[deref] view: View,
}
```

### Action 枚举

```rust
pub enum DesktopRunListAction {
    RunItem { mount: String, name: String },
    None,
}
```

### Row Data

```rust
enum RunListRowData {
    RunItem { mount: String, name: String },
    None,
}
```

## Widget trait 实现

### `draw_walk`
1. 委托 `view.draw_walk`
2. 在 `PortalList` step 回调中：
   - 从 scope data 获取 `AppData`
   - 如果没有 active mount，显示 "Select a mount"
   - 如果 run_items 为空，显示 "No run items available"
   - 渲染每个 runnable item：设置按钮文本 + `RunListRowData::RunItem` action data

### `handle_event`
监听 `PortalList` 中 `row_button` 的点击事件，读取 `RunListRowData` 并转发为 `DesktopRunListAction::RunItem`。

## DesktopRunListRef 方法

### `run_requested`
从 actions 中提取 `DesktopRunListAction::RunItem`，返回 `(mount, name)` 元组。

## 与 Hub 的交互

- 数据源：`AppData.mounts[active_mount].run_items`（由 `HubToClient::RunItems` 消息更新）
- 点击触发：`App` 收到 `RunItem` action 后发送 `ClientToHub::RunItem`
