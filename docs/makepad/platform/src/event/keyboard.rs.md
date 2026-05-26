# `keyboard.rs` — 键盘焦点与事件类型

## 概述

该文件定义了两个核心领域：**键盘焦点管理系统**（`CxKeyboard`）和**若干键盘相关事件数据结构**。键盘焦点的管理遵循"下一次焦点"模式，即通过 `next_key_focus` 暂存即将设置的焦点，然后在帧更新周期中统一切换到 `key_focus`。

---

## `CxKeyboard` — 键盘焦点管理器

```rust
pub struct CxKeyboard {
    prev_key_focus: Area,    // 上一次的焦点区域
    next_key_focus: Area,    // 下一次的焦点区域（暂存值）
    key_focus: Area,         // 当前实际的焦点区域
    keys_down: Vec<KeyEvent>, // 当前按下的按键列表
    text_ime_dismissed: bool, // IME 是否已被关闭
}
```

### 设计模式：双重缓冲

键盘焦点使用"暂存→提交"的两阶段模式：
1. **设置阶段**：调用 `set_key_focus(area)` 将目标焦点写入 `next_key_focus`
2. **提交阶段**：调用 `cycle_key_focus_changed()` 将 `next_key_focus` 提升为 `key_focus`，旧焦点存入 `prev_key_focus`
3. 这种设计确保在事件处理的同一帧内，多次设置焦点不会互相干扰

### 方法详解

#### `CxKeyboard::modifiers() -> KeyModifiers`
返回当前按下按键列表中的第一个事件的修饰键状态。如果无按键按下，返回 `Default::default()`。

#### `CxKeyboard::set_key_focus(focus_area)`
设置下一次帧更新时的目标焦点区域：
1. 重置 `text_ime_dismissed = false`
2. 将 `focus_area` 写入 `next_key_focus`

#### `CxKeyboard::key_focus() -> Area`
返回当前生效的焦点区域（`key_focus`）。

#### `CxKeyboard::revert_key_focus()`
回退焦点至上一次的区域：将 `next_key_focus` 设为 `prev_key_focus`。

#### `CxKeyboard::has_key_focus(focus_area) -> bool`
判断指定的区域是否为当前焦点区域。

#### `CxKeyboard::set_text_ime_dismissed()`
标记 IME 已被关闭（`text_ime_dismissed = true`）。

#### `CxKeyboard::reset_text_ime_dismissed()`
重置 IME 关闭标记（`text_ime_dismissed = false`）。

#### `CxKeyboard::update_area(old_area, new_area)`
当区域重映射时，查找并替换 `key_focus`、`prev_key_focus`、`next_key_focus` 中的引用。此方法确保焦点在新旧区域映射后仍然有效。

#### `CxKeyboard::cycle_key_focus_changed() -> Option<(Area, Area)>`
提交焦点变更：
1. 如果 `next_key_focus != key_focus`：
   - 将当前 `key_focus` 存入 `prev_key_focus`
   - 将 `next_key_focus` 提升为 `key_focus`
   - 返回 `Some((prev, current))` — 旧焦点和新焦点
2. 如果没有变化，返回 `None`

#### `CxKeyboard::is_key_down(key_code) -> bool`
检查指定键码是否正在按下。遍历 `keys_down` 列表查找匹配项。

#### `CxKeyboard::process_key_down(key_event)`
记录按键按下事件。如果该键码已经在列表中则不重复添加（防止重复记录）。

#### `CxKeyboard::process_key_up(key_event)`
记录按键释放事件。从 `keys_down` 列表中移除匹配的键码。

---

## `KeyFocusEvent`

```rust
pub struct KeyFocusEvent {
    pub prev: Area,   // 之前的焦点区域
    pub focus: Area,  // 新的焦点区域
}
```
键盘焦点变更时发送。`Event::KeyFocus` 和 `Event::KeyFocusLost` 使用此结构。

---

## `TextClipboardEvent`

```rust
pub struct TextClipboardEvent {
    pub response: Rc<RefCell<Option<String>>>,
}
```
文本剪贴板事件的响应容器。使用 `Rc<RefCell<Option<String>>>` 实现共享可变性：
- 请求方创建事件，设置 `response`
- 处理方读取或写入剪贴板内容
- 使用 `Rc` 允许多个引用，`RefCell` 提供运行时借用检查

---

## `TextRangeReplaceEvent`

```rust
pub struct TextRangeReplaceEvent {
    pub start: usize,   // 起始字符索引（非字节）
    pub end: usize,     // 结束字符索引（非字节）
    pub text: String,   // 要插入的文本
}
```
用于替换文本指定范围的专用事件。`start` 和 `end` 是字符索引（Unicode 标量值计数），而非字节偏移，确保正确处理多字节字符。

---

## 重新导出的类型

文件从 `makepad_studio_protocol` 重导出了以下重要类型：

| 类型 | 用途 |
|------|------|
| `KeyCode` | 按键码枚举 |
| `KeyEvent` | 按键事件（包含 `key_code`、`modifiers`、`char` 等） |
| `TextInputEvent` | 文本输入事件 |
| `ImeActionEvent` | IME 动作事件 |
| `ImeAction` | IME 动作类型枚举 |
| `CharOffset` | 字符偏移类型 |
| `FullTextState` | 全文状态快照 |
