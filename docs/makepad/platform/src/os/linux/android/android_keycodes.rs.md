# Android 键码映射

## 概述

`android_keycodes.rs` 定义了 Android 键码到 Makepad `KeyCode` 枚举的映射。

## 参考数据

文件顶部包含从 Android 源码 [KeyEvent.java](https://android.googlesource.com/platform/frameworks/base/+/95c1165/core/java/android/view/KeyEvent.java) 提取的完整键码枚举注释（共 300+ 条目）。这些注释作为离线参考，未在代码中直接使用。

## 核心函数

### `android_to_makepad_key_code(key_code: u32) -> KeyCode`

将 Android 原始键码（`u32`）映射到 Makepad `KeyCode` 枚举。

```rust
KeyCode::Back        ← 4
KeyCode::Key0-9      ← 7-16
KeyCode::ArrowUp/Down/Left/Right ← 19-22
KeyCode::KeyA-Z      ← 29-54
KeyCode::ReturnKey   ← 66
KeyCode::Backspace   ← 67
KeyCode::Escape      ← 111
KeyCode::Delete      ← 112
KeyCode::F1-F12      ← 131-142
KeyCode::Numpad0-9   ← 144-153
// 小键盘符号键
// 修饰键（左右 Shift/Alt/Ctrl 都映射到同一 KeyCode）
```

## 特殊映射说明

| Android 键码 | 映射到 | 说明 |
|-------------|--------|------|
| 57（左 Alt）| `KeyCode::Alt` | 左右 Alt 合并 |
| 58（右 Alt）| `KeyCode::Alt` | |
| 59（左 Shift）| `KeyCode::Shift` | 左右 Shift 合并 |
| 60（右 Shift）| `KeyCode::Shift` | |
| 113（左 Ctrl）| `KeyCode::Control` | 左右 Ctrl 合并 |
| 114（右 Ctrl）| `KeyCode::Control` | |
| 158 | `KeyCode::NumpadDecimal` | `NumpadComma` 也映射到 `NumpadDecimal` |
| 未匹配项 | `KeyCode::Unknown` | 默认处理 |

## 实现说明

- `pub(crate)` 可见性，仅在 crate 内部使用。
- 无修饰键状态信息在此处理（修饰键在 `android.rs` 的 `FromJavaMessage::KeyDown`/`KeyUp` 处理中管理）。
- 参考注释包含完整的 Android 键码枚举（0-304），但映射函数只覆盖了 Makepad 需要的子集。
