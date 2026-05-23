# link_label.rs — 可点击链接标签

## 概述
`LinkLabel` 是 `Button` 的包装组件，专门用于打开 URL 链接。当被点击时，调用 `cx.open_url` 在系统浏览器中打开预设 URL。支持 `open_in_place` 决定是否在同一窗口打开。

## 核心结构

### LinkLabel
- **`button: Button`**：通过 `#[deref]` 委托给 Button，复用其所有交互逻辑。
- **`url: String`**：点击时打开的 URL。
- **`open_in_place: bool`**：是否在同一位置打开（Open URL in place）。

## 核心方法

### Widget 实现

**`script_call`**：处理脚本层的方法调用 — `text()` 返回当前标签文本，`set_text(text)` 设置新文本。

**`handle_event`**：通过 `cx.capture_actions` 捕获 Button 事件，检测到点击且 URL 非空时调用 `cx.open_url` 打开链接。然后将捕获的动作通过 `cx.extend_actions` 传递出去。

**`draw_walk`**：直接委托给内部 Button 的 `draw_walk`。

**`text`/`set_text`**：委托给 Button 的对应方法。

### 动作检测

**`clicked`/`pressed`/`released`**：检测链接是否被点击/按下/释放。

**`clicked_modifiers`/`pressed_modifiers`/`released_modifiers`**：返回携带键盘修饰键的点击/按下/释放事件。

### LinkLabelRef

`LinkLabelRef` 提供所有方法的委托版本，使用 `borrow`/`borrow_mut` 安全访问内部 `LinkLabel`。
