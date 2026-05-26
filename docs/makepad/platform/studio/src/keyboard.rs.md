# keyboard.rs — 键盘事件类型

## 概述

`keyboard.rs` 定义了 Makepad 平台层的键盘事件数据结构，包括键码枚举、按键事件、文本输入事件、IME（输入法）相关类型，以及字符偏移工具。所有类型均支持序列化。

---

## KeyEvent

表示一次键盘按键事件（按下或释放）。

```rust
#[derive(Clone, Copy, Debug, Default, SerBin, DeBin, SerJson, DeJson, PartialEq)]
pub struct KeyEvent {
    pub key_code: KeyCode,     // 按键的键码
    pub is_repeat: bool,       // 是否来自键盘重复
    pub modifiers: KeyModifiers, // 同时按下的修饰键
    pub time: f64,              // 事件时间戳
}
```

---

## TextInputEvent

表示文本输入事件（来自 IME、粘贴、键盘直接输入等）。

```rust
#[derive(Clone, Debug, PartialEq)]
pub struct TextInputEvent {
    pub input: String,                              // 输入的文本
    pub replace_last: bool,                          // 是否替换上一次输入
    pub was_paste: bool,                             // 是否来自粘贴操作
    pub composition: Option<Range<usize>>,           // IME 组合范围（字符偏移，在 input 字符串内）
    pub full_state_sync: Option<FullTextState>,      // 完整文本状态同步（仅 Android）
    pub replace_range: Option<(CharOffset, CharOffset)>, // 替换范围（iOS 自动纠正/粘贴）
}
```

### 序列化策略

`TextInputEvent` 使用手动序列化实现：仅序列化 `input`、`replace_last`、`was_paste` 三个"线兼容"字段。IME 专用字段（`composition`、`full_state_sync`、`replace_range`）仅在进程内使用，不通过 stdin 协议传输。

中间结构体：
```rust
struct TextInputEventWire {
    input: String,
    replace_last: bool,
    was_paste: bool,
}
```

手动实现了 `SerBin`、`DeBin`、`SerJson`、`DeJson`，序列化时构造 `TextInputEventWire` 转发，反序列化时回填到 `TextInputEvent` 并用 `Default::default()` 填充 IME 字段。

---

## CharOffset — Unicode 字符偏移

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, SerBin, DeBin, SerJson, DeJson)]
pub struct CharOffset(pub usize);
```

一个平台无关的文本位置索引类型，基于 Unicode 标量值（char）计数，而非字节或 UTF-16 代码单元。

### 方法

| 方法 | 说明 | 参数/返回值 |
|------|------|-------------|
| `to_byte_index(text)` | 转换为 UTF-8 字节索引 | 返回 `usize` |
| `from_utf16_index(text, utf16_idx)` | 从 UTF-16 索引转换（Android/Java） | 返回 `CharOffset` |
| `to_utf16_index(self, text)` | 转换为 UTF-16 索引（Android/Java） | 返回 `usize` |
| `range_to_bytes(range, text)` | 将 `Range<CharOffset>` 转换为 `Range<usize>`（字节偏移） | 静态方法 |

---

## FullTextState — 完整文本状态（IME）

```rust
#[derive(Clone, Debug, PartialEq)]
pub struct FullTextState {
    pub text: String,                          // 文本框中的完整文本
    pub selection: Range<CharOffset>,           // 当前选区
    pub composition: Option<Range<CharOffset>>, // 当前组合范围
}
```

专门用于 Android `InputConnection` 的文本状态同步。

---

## ImeAction — 输入法编辑器动作

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ImeAction {
    Unspecified,  // 未指定
    None,         // 无操作
    Go,           // 前往
    Search,       // 搜索
    Send,         // 发送
    Next,         // 下一个
    Done,         // 完成
    Previous,     // 上一个
}
```

### ImeAction 方法

```rust
/// 从 Android EditorInfo.actionId 转换
pub fn from_android_action_code(code: i32) -> ImeAction
```

| Action ID | ImeAction |
|-----------|-----------|
| 0 | Unspecified |
| 1 | None |
| 2 | Go |
| 3 | Search |
| 4 | Send |
| 5 | Next |
| 6 | Done |
| 7 | Previous |

### ImeActionEvent

```rust
#[derive(Clone, Debug)]
pub struct ImeActionEvent {
    pub action: ImeAction,
}
```

---

## KeyCode — 键码枚举

```rust
#[derive(Script, ScriptHook, Clone, Copy, Debug, SerBin, DeBin, Eq, PartialEq)]
pub enum KeyCode {
    // 功能键
    Escape, Back, Backtick,
    // 数字行
    Key0..=Key9, Minus, Equals,
    // 编辑键
    Backspace, Tab,
    // 第一行字母
    KeyQ..=KeyP, LBracket, RBracket, ReturnKey,
    // 第二行字母
    KeyA..=KeyL, Semicolon, Quote, Backslash,
    // 第三行字母
    KeyZ..=KeyM, Comma, Period, Slash,
    // 修饰键
    Control, Alt, Shift, Logo,
    // 空白与锁定
    Space, Capslock,
    // 功能键 F1-F12
    F1..=F12,
    // 系统键
    PrintScreen, ScrollLock, Pause,
    // 导航键
    Insert, Delete, Home, End, PageUp, PageDown,
    // 数字键盘
    Numpad0..=Numpad9, NumpadEquals, NumpadSubtract, NumpadAdd,
    NumpadDecimal, NumpadMultiply, NumpadDivide, Numlock, NumpadEnter,
    // 方向键
    ArrowUp, ArrowDown, ArrowLeft, ArrowRight,
    // 未知
    Unknown,
}
```

共 102 个变体（包括 `Unknown`）。

### KeyCode 方法

#### `is_unknown() -> bool`
检查是否为未知键。

#### `to_char(uc: bool) -> Option<char>`
将字母/数字/符号键转换为对应的字符。`uc=true` 时返回大写/上档字符。

| 键码 | `false` | `true` |
|------|---------|--------|
| KeyA | `'a'` | `'A'` |
| Key0 | `'0'` | `')'` |
| Minus | `'-'` | `'_'` |
| Equals | `'='` | `'+'` |
| LBracket | `'['` | `'{'` |
| RBracket | `']'` | `'}'` |
| Backslash | `'\\'` | `'\|'` |
| Semicolon | `';'` | `':'` |
| Quote | `'\''` | `'"'` |
| Comma | `','` | `'<'` |
| Period | `'.'` | `'>'` |
| Slash | `'/'` | `'?'` |
| Backtick | `` '`' `` | `'~'` |
| Space | `' '` | `' '` |
| Tab | `'\t'` | `'\t'` |
| ReturnKey / NumpadEnter | `'\n'` | `'\n'` |
| NumpadDecimal | `'.'` | `'.'` |
| NumpadMultiply | `'*'` | `'*'` |
| NumpadAdd | `'+'` | `'+'` |
| NumpadDivide | `'/'` | `'/'` |
| NumpadSubtract | `'-'` | `'-'` |
| Numpad0..=Numpad9 | `'0'`-`'9'` | `'0'`-`'9'` |

### JSON 序列化优化

`KeyCode` 使用手动 `SerJson`/`DeJson` 实现，通过整数索引替代派生宏的字符串匹配。这是因为 Rust 为 80+ 变体的枚举生成的字符串匹配的 LLVM IR 约 9500 行，而整数编码方案仅约 100 行：

```rust
const KEYCODE_VARIANTS: [KeyCode; 102] = [ /* 所有变体按序排列 */ ];

impl SerJson for KeyCode {
    fn ser_json(&self, _d: usize, s: &mut SerJsonState) {
        let idx = KEYCODE_VARIANTS.iter().position(|k| k == self).unwrap_or(101);
        s.out.push_str(&idx.to_string());
    }
}

impl DeJson for KeyCode {
    fn de_json(s: &mut DeJsonState, i: &mut std::str::Chars) -> Result<Self, DeJsonErr> {
        let val = u64::de_json(s, i)? as usize;
        Ok(KEYCODE_VARIANTS.get(val).copied().unwrap_or(KeyCode::Unknown))
    }
}
```
