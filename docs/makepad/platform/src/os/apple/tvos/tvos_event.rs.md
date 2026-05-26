# tvos_event.rs — tvOS 事件类型

**文件路径:** `platform/src/os/apple/tvos/tvos_event.rs`

**核心目的:** 定义 tvOS 平台事件枚举，用于将 Apple 远程控制/Siri Remote 输入映射到 Makepad 事件系统。

```rust
pub enum TvosEvent {
    Menu,
    PlayPause,
    Select,
    Swipe(SwipeDirection),
    Siri,
}
```

**实现细节:**
- `TvosEvent` 枚举表示 Apple TV 遥控器的物理按键和手势
- `SwipeDirection` 是内部枚举，包含 `Up`, `Down`, `Left`, `Right`
- 文件简短（~14 行），专注于事件类型的定义和派生属性 (`Debug, Clone, Copy, PartialEq`)

**平台集成:** tvOS 专用，通过 tvOS 应用代理从 `GCEventViewController` 的按键/手势回调中创建
