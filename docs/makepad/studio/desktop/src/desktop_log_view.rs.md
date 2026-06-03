# `desktop_log_view.rs` — Studio 日志视图组件

## 文件作用

在底部面板中显示构建日志，支持日志级别图标、行号/文件位置跳转链接、自动跟随滚动和选择。

## script_mod 定义

### 子组件

**`LogIcon`**: 10×10 的彩色图标容器，支持三种变体：
- `log_icon`：灰色圆形（普通日志）
- `warning_icon`：黄色三角形（警告，SDF 三⻆形 + ! 标记）
- `error_icon`：红色圆形 + X 标记（错误/恐慌）

**`LogItem`**: 日志条目行（View）：
- `height: Fit`（高度自适应）
- `TextFlow` 容器（`selectable: true`）
- `CodeView` 编辑器区域（`margin.left: 25`）
- `FoldButton` 折叠按钮
- 三种日志图标（`log_icon`, `warning_icon`, `error_icon`），根据日志级别切换显示
- `Animator`：hover（渐变色）和 select（选中高亮）动画

**`LogEmptyItem`**: 空日志占位项（32px 高度）。

**`DesktopLogView`**: 完整日志组件：
```
list := PortalList {
    auto_tail: true
    selectable: true
    LogItem := LogItem (注意：使用大写 L 的 ID)
    Empty := LogEmptyItem
}
```

## 结构体

```rust
#[derive(Script, Widget)]
pub struct DesktopLogView {
    #[deref] view: View,
    #[rust] tail: bool,  // 是否自动跟随底部
}
```

默认 `tail = true`（在 `ScriptHook::on_after_new` 中设置）。

### Action 枚举

```rust
pub enum DesktopLogViewAction {
    OpenLocation { path: String, line: usize, column: usize },
    None,
}
```

### LogLocationLink

```rust
struct LogLocationLink {
    path: String,
    line: usize,
    column: usize,
}
```

用作 `TextFlow` 中的可点击链接数据。

## Widget trait 实现

### `draw_walk`
1. 获取当前 tab_id（从 widget tree 路径）
2. 委托 `view.draw_walk`
3. 在 `PortalList` step 回调中：
   - 从 scope data 获取 `AppData`
   - 调用 `collect_entries` 获取日志条目（优先 build 日志 → mount 日志）
   - 调用 `draw_entries` 渲染

### `draw_entries`
对每个可见日志条目：
1. 设置交替背景色
2. 委托 item 的 `draw` 方法
3. 在 step 回调中操作 `TextFlow`：
   - `draw_item_counted` — 根据日志级别显示对应的图标（通过 count 参数控制，仅渲染一个图标）
   - `draw_link` — 如果存在 `UiLogLocation`，渲染为可点击链接
   - `draw_text` — 日志消息文本

**关键模式**: 使用 `TextFlow` 在同一行内绘制图标、链接和文本，实现类似终端日志的紧凑布局。

### `handle_event`
监听 `PortalList` 中 `LogLocationLink` 类型的 action（链接点击）：
- 转发为 `DesktopLogViewAction::OpenLocation`

## DesktopLogViewRef 方法

- `set_tail`: 启用/禁用自动跟随。设置 `PortalList` 的 `tail_range` 并在启用时滚动到底部。
- `tail`: 返回当前跟随状态。
- `scrolled`: 检查是否发生了滚动事件。
- `is_at_end`: 检查是否在列表底部。
- `open_location_requested`: `(path, line, column)` 提取方法。

## 日志级别→图标映射

- `LogLevel::Error | Panic` → `error_icon`（红色圆形 + X）
- `LogLevel::Warning | Wait` → `warning_icon`（黄色三角形 + !）
- `LogLevel::Log` → `log_icon`（灰色圆形）

## 与 Hub 的交互

日志条目在 `app_messages.rs` 的 `QueryLogResults` 处理中更新。Hub 发送的原始 `LogEntry` 被转换为 `UiLogEntry`，其中的 `location` 字段通过 `extract_log_location` 从日志消息中解析。
