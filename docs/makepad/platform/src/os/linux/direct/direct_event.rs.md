# Linux Direct 事件枚举

## 概述

`direct_event.rs` 定义了 Linux Direct 平台的事件枚举类型。它是 Makepad 核心事件类型和 Linux 原始输入之间的桥梁。

## 类型

### `DirectEvent`

```rust
pub enum DirectEvent {
    Paint,                          // 渲染事件
    MouseDown(MouseDownEvent),      // 鼠标按下
    MouseUp(MouseUpEvent),          // 鼠标抬起
    MouseMove(MouseMoveEvent),      // 鼠标移动
    Scroll(ScrollEvent),            // 滚轮事件
    KeyDown(KeyEvent),              // 按键按下
    KeyUp(KeyEvent),                // 按键抬起
    TextInput(TextInputEvent),      // 文本输入
    Timer(TimerEvent),              // 定时器事件
}
```

## 说明

- 所有字段类型引用自 `crate::event` 模块，与 Makepad 核心事件系统共享定义。
- 事件由 `raw_input.rs` 或 `linux_direct.rs` 中的定时器生成，在 `linux_direct.rs::direct_event_callback` 中处理。
- `MouseDownEvent`/`MouseUpEvent`/`MouseMoveEvent`/`ScrollEvent` 使用 `Cell<Area>` 的 `handled` 字段实现事件冒泡。
- `KeyEvent` 同时用于 `KeyDown` 和 `KeyUp`，通过 `is_repeat` 字段区分重复按键。

## 实现说明

- 此文件仅为枚举定义，不含任何逻辑实现。
- `TimerEvent` 由 `SelectTimers` 系统生成，关联 `timer_id` 和 `time`。
