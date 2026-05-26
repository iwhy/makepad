# macos_event.rs — macOS 事件处理

**文件路径:** `platform/src/os/apple/macos/macos_event.rs`

**核心目的:** 将 macOS 原生 `NSEvent` 转换为 Makepad 内部事件格式。包含键盘、鼠标、滚动、触摸条（Touch Bar）和拖放事件的完整映射。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `MacosKeyMapping` | 键盘映射表结构体，包含虚拟键码到 Makepad 键名的映射 |
| `MacosEventConverter` | NSEvent → MakepadEvent 转换器，支持输入状态跟踪 |

**关键方法:**
- `MacosKeyMapping::new()` — 初始化标准 QWERTY 映射表
- `MacosKeyMapping::virtual_key_code_to_str(key)` — 虚拟键码 → Makepad 键名字符串
- `MacosKeyMapping::modifier_to_makepad(modifiers)` — NSEvent 修饰符掩码 → Makepad `KeyModifiers`
- `convert_event(event, key_mapping)` — 将 NSEvent 转换为 `Option<MakepadEvent>`，处理：
  - `NSKeyDown`/`NSKeyUp`: keys 键 → 字符键映射，支持 IME 输入
  - `NSFlagsChanged`: 修饰键（Shift/Ctrl/Alt/Cmd）状态变更
  - `NSMouseMoved`/`NSLeftMouseDragged` 等: 鼠标位置和移动
  - `NSScrollWheel`: 滚动事件（精确滚动和像素滚动）
  - `NSLeftMouseDown/Up`: 鼠标点击
  - `NSTabletPoint`/`NSTabletProximity`: 数位板/手写笔事件（压力、倾斜、旋转）
  - `NSEventTypeGesture`/`Swipe`: 触摸板手势

**按键映射实现细节:**
- 使用 128 元素的 `[Option<&str>; 128]` 静态数组存储虚拟键码映射
- 特殊处理 ANSI 'A' 和 ANSI 'B'（区分 QWERTY/AZERTY 布局）
- IME 输入支持：通过 `characters` 方法获取已转换的字符
- 修饰键合成：CapsLock 状态通过 `-[NSEvent modifierFlags]` 检测

**手势处理:**
- `NSEventTypeBeginGesture`/`EndGesture`: 手势开始/结束标记
- `NSEventTypeSwipe`: 三/四指轻扫（通过 `eventSwipe` 扩展方法）
- `NSEventTypeMagnify`: 双指缩放
- `NSEventTypeRotate`: 双指旋转

**平台集成:** macOS 专用，使用 AppKit 的 NSEvent API
