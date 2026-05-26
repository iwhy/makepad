# ios_text_input.rs — UITextInput 协议实现

**文件路径**: `platform/src/os/apple/ios/ios_text_input.rs` (2161 行)
**核心作用**: 完整实现 iOS `UITextInput` 协议（`UIKeyInput` + `UITextInput` + `UITextInputTraits`），提供 IME 输入法支持（中日韩文组合输入）、光标定位、文本选择、自动纠正、听写、浮动光标（键盘触控板）功能。

## 架构概览

该模块使用 Rust 动态注册三个关键 Objective-C 类：
1. **MakepadTextPosition** — 自定义 `UITextPosition` 子类
2. **MakepadTextRange** — 自定义 `UITextRange` 子类
3. **MakepadSelectionRect** — 自定义 `UITextSelectionRect` 子类
4. **MakepadTextInputView** — 核心类，继承 `UIView`，遵循 `UIKeyInput` + `UITextInput` + `UITextInputTraits` 协议

## UTF-16 索引辅助函数

iOS NSString 使用 UTF-16 编码，而 Makepad 使用字符索引。

| 函数 | 说明 |
|------|------|
| `utf16_len(s)` | Rust 字符串的 UTF-16 码元计数 |
| `utf16_indices_to_char_offsets(text, utf16_start, utf16_end)` | UTF-16 索引 → 字符索引对（单遍扫描） |
| `char_to_utf16_index(text, char_index)` | 字符索引 → UTF-16 索引 |
| `char_index_to_byte_index(text, char_index)` | 字符索引 → 字节索引 |
| `utf16_range_to_string(text, start, end)` | UTF-16 范围提取子字符串 |

## MakepadTextPosition

继承 `UITextPosition`，存储单个 ivar `_offset: i64`（UTF-16 偏移量）。

| 方法 | 作用 |
|------|------|
| `offset` / `setOffset:` | 获取/设置偏移 |
| `positionWithOffset:` | 类工厂方法，创建并 autorelease 新实例 |

## MakepadTextRange

继承 `UITextRange`，存储 `_startOffset` 和 `_endOffset` 而非位置对象引用。

| 方法 | 作用 |
|------|------|
| `start` / `end` | 从偏移量创建并返回位置对象 |
| `isEmpty` | start == end |
| `rangeWithStart:end:` | 类工厂方法（偏移量） |
| `rangeWithStartPosition:endPosition:` | 类工厂方法（位置对象） |

## MakepadSelectionRect

继承 `UITextSelectionRect`，用于 iOS 16+ `UITextSelectionDisplayInteraction`。

| ivar | 说明 |
|------|------|
| `x/y/w/h` | 矩形坐标 |
| `contains_start/contains_end` | 是否包含选择起点/终点 |

| 方法 | 作用 |
|------|------|
| `rect` | 返回 NSRect |
| `containsStart` / `containsEnd` | 端点包含标志 |
| `rectWithX:y:w:h:containsStart:containsEnd:` | 类工厂方法 |

## MakepadTextInputView

核心 IME 类，继承 `UIView`，约 30+ ivar 和 50+ 方法。

### 实例变量

**文本状态**:
| ivar | 类型 | 说明 |
|------|------|------|
| `markedText` | `ObjcId` (NSMutableAttributedString) | 激活的组合文本 |
| `markedTextStart` | `i64` | 组合文本起始 UTF-16 偏移 |
| `textBuffer` | `ObjcId` (NSMutableString) | 当前文本缓冲区 |
| `cursorPosition` | `i64` | 光标位置 |
| `selectionStart/End` | `i64` | 选择范围 |

**键盘配置（UITextInputTraits）**:
| ivar | 类型 | 说明 |
|------|------|------|
| `_keyboard_type` | `i64` | UIKeyboardType |
| `_autocapitalization_type` | `i64` | UITextAutocapitalizationType |
| `_autocorrection_type` | `i64` | -1 表示 CJK 自动检测 |
| `_return_key_type` | `i64` | UIReturnKeyType |
| `_secure_text_entry` | `bool` | 安全文本输入 |

**浮动光标**:
| ivar | 说明 |
|------|------|
| `floating_cursor_active` | 浮动光标激活标志 |
| `floating_cursor_last_x/y` | 上次位置 |

**选择显示**:
| ivar | 说明 |
|------|------|
| `selection_handle_start_x/y` / `end_x/y` | 选择手柄锚点 |
| `selection_handles_visible` | 选择显示标志 |

### UIResponder

- `canBecomeFirstResponder` → `YES`

### UIKeyInput 协议

- **`hasText`** — textBuffer 或 markedText 非空
- **`insertText:`** — 主要文本插入入口：
  1. `\n` 发送 Return 按键事件（不由 insertText 处理文本）
  2. 检查如果有激活的组合文本，使用 markedTextStart 作为插入位置
  3. 无组合文本时使用当前选择范围
  4. 如果选择范围非空（如自动纠正），发送 `TextRangeReplace` 事件而非普通 `TextInput`
  5. 更新 textBuffer（`replaceBufferRange`）
  6. 通过 inputDelegate 发送 `textWillChange`/`selectionWillChange`/`textDidChange`/`selectionDidChange` 通知
- **`deleteBackward`** — 删除逻辑：
  1. 有选择范围 → 发送 `TextRangeReplace`（空字符串）
  2. 光标前删除 → 正确处理多码元 UTF-16 字符（如 emoji），送 `KeyEvent(Backspace)`
  3. 恢复越界光标

### UITextInput — 组合文本（Marked Text）

- **`hasMarkedText`** — 检查 markedText 是否非空
- **`markedTextRange`** — 返回组合文本的 UITextRange
- **`setMarkedText:selectedRange:`** — 设置组合文本：
  1. 如果之前无组合文本，删除当前选择范围
  2. 存储 markedText 和 markedTextStart
  3. 发送 `TextInput(marked_string, true)`（`replace_last = true` 替换上次组合预览）
  4. 更新所选范围到组合文本内
- **`unmarkText`** — 确认组合文本：
  1. 发送 `TextInput(string, false)` 提交文本
  2. 插入到 textBuffer
  3. 清除 markedText

### UITextInput — 选择管理

- **`selectedTextRange`** / **`setSelectedTextRange:`** — 获取/设置选择范围
- 选择变化时通过 `send_selection_changed` 排队 `SelectionChanged` 事件
- 检测变化避免不必要的通知

### UITextInput — 文本存储

- **`textInRange:`** — 从可见文本（含组合文本）中提取子串
- **`attributedTextInRange:`** — 同上，返回 NSAttributedString
- **`replaceRange:withText:`** — 范围替换：
  1. 清除组合文本
  2. 更新 textBuffer
  3. 转换 UTF-16 索引为字符索引
  4. 发送 `TextRangeReplace` 事件
- **`shouldChangeTextInRange:replacementText:`** → `YES`

### UITextInput — 位置/范围

完整实现 UITextInput 位置/范围 API（约 15 个方法）：
- `beginningOfDocument` / `endOfDocument`（含组合文本长度）
- `positionFromPosition:offset:` / `positionFromPosition:inDirection:offset:`
- `textRangeFromPosition:toPosition:` / `comparePosition:toPosition:` / `offsetFromPosition:toPosition:`
- `positionWithinRange:farthestInDirection:` / `positionWithinRange:atCharacterOffset:`
- `characterOffsetOfPosition:withinRange:` / `characterRangeByExtendingPosition:inDirection:`
- 所有操作确保偏移量在 `[0, visible_text_utf16_len]` 范围内

### UITextInput — 几何

- **`firstRectForRange:`** — 返回 IME 位置的光标矩形（用于候选窗口定位）。选择手柄可见时返回选择范围包围盒
- **`caretRectForPosition:`** — 选择可见时根据偏移选择起点/终点手柄位置，否则回退到 firstRectForRange
- **`selectionRectsForRange:`** — 选择可见时返回单个 MakepadSelectionRect，否则空数组
- **`closestPositionToPoint:`** — 返回当前光标位置（精确命中测试需要 widget 层布局信息）
- **`characterRangeAtPoint:`** / **`unobscuredContentRect`** / **`textInputView`**

### UITextInput — 书写方向

- `baseWritingDirectionForPosition:inDirection:` → `NSWritingDirectionNatural`
- `setBaseWritingDirection:forRange:` — 空操作

### UITextInput — 委托和分词器

- `inputDelegate` / `setInputDelegate:` — 管理 `id<UITextInputDelegate>`
- `tokenizer` — 懒初始化 `UITextInputStringTokenizer`

### UITextInputTraits

| 方法 | 实现 |
|------|------|
| `keyboardType` | 从 ivar 读取 |
| `autocorrectionType` | CJK 自动检测：当 `_autocorrection_type == -1` 时检查键盘语言，zh/ja/ko 返回 `NO` |
| `autocapitalizationType` | 从 ivar 读取 |
| `spellCheckingType` | `Default` |
| `smartQuotesType` / `smartDashesType` / `smartInsertDeleteType` | 全部 `No`（用于代码/文本编辑） |
| `returnKeyType` | 从 ivar 读取 |
| `isSecureTextEntry` | 从 ivar 读取 |
| `enablesReturnKeyAutomatically` | `NO` |

### 听写（Dictation）

- **`insertDictationResult:`** — 遍历 UITextRange 数组，处理 NSString、UITextPhraseAlternatives 等，合并结果为文本后调用 `insertText`
- **`insertDictationResultPlaceholder`** / **`frameForDictationResultPlaceholder:`** / **`removeDictationResultPlaceholder:`** — 占位符管理

### 浮动光标（Keyboard Trackpad）

浮动光标将手指在键盘区域的滑动转换为方向键事件：

- **`beginFloatingCursorAtPoint:`** — 激活并记录起始位置
- **`updateFloatingCursorAtPoint:`** — 水平方向每 10pt 触发 ArrowLeft/Right，垂直方向每 20pt 触发 ArrowUp/Down，通过 `IosApp::do_callback(KeyDown/Up)` 发送
- **`endFloatingCursor`** — 停用

### 协议注册

```rust
if let Some(protocol) = Protocol::get("UIKeyInput") {
    decl.add_protocol(protocol);
}
if let Some(protocol) = Protocol::get("UITextInput") {
    decl.add_protocol(protocol);
}
```

MakepadTextInputView 声明遵循 `UIKeyInput` 和 `UITextInput` 协议，允许 iOS 系统将其识别为有效的文本输入视图。
