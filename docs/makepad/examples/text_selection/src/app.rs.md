# text_selection/src/app.rs

演示 Makepad 的文本选择功能，包括可选/不可选文本和输入框文本选择。

## 整体结构

- **第5-117行**：`script_mod!` UI 定义

### 三个演示区域

1. **可选文本**（第31-57行）：`Markdown` 组件设置 `selectable: true`，支持鼠标拖拽或移动端长按选择文本
2. **可选输入框**（第59-85行）：`TextInput` 组件（`is_multiline: true`），输入后可以部分选中文字
3. **不可选文本**（第87-111行）：
   - `Label` 组件默认不可选
   - `Markdown` 组件设置 `selectable: false`，拖拽不应产生选区

### Rust 侧（第120-134行）

- App 结构体
- `AppMain` 标准实现，`handle_event` 仅将事件传递给 widget 树

## 关键 API

- `Markdown { selectable: true/false }` — 控制 Markdown 文本是否可选
- `TextInput` — 输入框文本选择（输入后光标选择）
- 移动端通过长按触发选择手柄
- 桌面端通过鼠标拖拽选中文本
