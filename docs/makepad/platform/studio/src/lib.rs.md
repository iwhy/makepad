# lib.rs — platform/studio 库入口

## 概述

`lib.rs` 是 `makepad-platform-studio` crate 的根模块。它重新导出所有公共类型和子模块，供外部 crate（如 `makepad-widgets` 中的 Studio 集成代码）使用。

## 子模块

```rust
pub mod cursor;           // MouseCursor 枚举
pub mod hub_protocol;     // Hub <-> Client 协议消息
pub mod keyboard;         // 键盘事件类型
pub mod mouse;            // 鼠标事件类型
pub mod shared_framebuf;  // 共享帧缓冲/交换链
pub mod studio;           // 核心协议数据类型 (AppToStudio, StudioToApp 等)
```

## 重新导出的类型

### 来自 cursor
```rust
pub use cursor::MouseCursor;
```

### 来自 keyboard
```rust
pub use keyboard::{
    CharOffset,        // Unicode 字符偏移
    FullTextState,     // 完整文本状态（IME 用）
    ImeAction,         // IME 编辑器动作
    ImeActionEvent,    // IME 动作事件
    KeyCode,           // 键码枚举
    KeyEvent,          // 键盘事件
    TextInputEvent,    // 文本输入事件
};
```

### 来自 mouse
```rust
pub use mouse::{KeyModifiers, MouseButton};
```

### 来自 shared_framebuf
所有公共类型通过通配符 `*` 重新导出。

### 来自 studio
所有公共类型通过通配符 `*` 重新导出，包括：
- `AppToStudio` / `StudioToApp` 枚举
- 所有请求/响应结构体（ScreenshotRequest, WidgetTreeDumpRequest 等）
- 远程输入事件结构体
- 性能分析样本结构体
- 编辑器操作结构体

### 来自 makepad_error_log
```rust
pub use makepad_error_log::LogLevel;
```

## 使用方式

外部 crate 可以通过以下方式导入：

```rust
use makepad_platform_studio::{
    AppToStudio, StudioToApp,
    KeyEvent, TextInputEvent, KeyCode,
    MouseButton, KeyModifiers,
    MouseCursor,
    ScreenshotRequest, WidgetTreeDumpRequest,
    // ... 等
};
```
