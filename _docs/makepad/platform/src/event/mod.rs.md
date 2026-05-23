# `mod.rs` — 事件系统模块入口

## 概述

该文件是整个 Makepad 事件系统的**模块入口文件**。它通过 `pub mod` 声明所有子模块，并通过 `pub use` 将所有子模块的公共类型重新导出到父命名空间中，从而对外提供统一的事件系统 API。

## 模块声明

每个 `pub mod` 语句声明一个子模块：

| 子模块 | 文件 | 职责 |
|--------|------|------|
| `drag_drop` | `drag_drop.rs` | 拖放事件处理（Drag & Drop） |
| `event` | `event.rs` | 核心事件枚举及辅助类型（Event, Hit, Ease 等） |
| `finger` | `finger.rs` | 手指/触摸/鼠标事件及命中测试系统 |
| `game_input` | `game_input.rs` | 游戏手柄和方向盘输入事件 |
| `keyboard` | `keyboard.rs` | 键盘焦点管理 |
| `network` | `network.rs` | 网络响应事件 |
| `video_playback` | `video_playback.rs` | 视频播放生命周期事件 |
| `window` | `window.rs` | 窗口几何/安全区变更事件 |
| `xr` | `xr.rs` | XR（扩展现实）输入事件 |

## 重新导出

每个 `pub use` 语句将子模块的所有公共类型提升到父模块作用域，这样外部代码只需要 `use crate::event::*` 即可访问所有事件类型，无需分别引用每个子模块。

## 设计意图

Makepad 事件系统采用模块拆分 + 统一导出的设计模式：
- **内部按职责拆分**：每个事件类别有独立的 .rs 文件，便于维护和扩展
- **外部单一入口**：通过 `pub use` 聚合，使用者只需引用 `event` 模块即可获得完整的类型覆盖
- **松散耦合**：`event.rs` 中的 `Event` 枚举成员引用其他子模块的类型（如 `DragEvent`、`KeyEvent`），事件处理逻辑（如命中测试）分散在各子模块中，通过 `impl Event` 注入方法
