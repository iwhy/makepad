# mod.rs — Web 平台子模块声明与重导出

**文件路径**: `/home/ubuntu/_github/makepad/platform/src/os/web/mod.rs` (14 行)
**核心作用**: 声明 Web 平台操作系统的各个子模块，并将核心类型重导出到 `crate::os::web` 命名空间。

## 模块声明

- `pub mod web` — 主平台集成（事件循环、消息桥接），标注 `#[macro_use]`
- `pub mod from_wasm` — Rust→JS 的消息类型（`FromWasm`）
- `pub mod to_wasm` — JS→Rust 的消息类型（`ToWasm`）
- `pub mod web_audio` — Web Audio API 集成
- `pub mod web_gl` — WebGL 渲染管线
- `pub mod web_media` — 媒体管理（音频/MIDI/视频）
- `pub mod web_midi` — Web MIDI API 集成
- `pub mod web_network` — 网络后端 Shim（HTTP + WebSocket）
- `mod web_socket` — WebSocket 实现（私有模块）

## 重导出

- `pub use crate::os::web::web::*` — 主平台所有公共项
- `pub use crate::os::web::web_gl::*` — WebGL 渲染所有公共项
- `pub use crate::os::web::web_midi::*` — MIDI 所有公共项
- `pub use crate::os::web::web_socket::OsWebSocket` — WebSocket 结构体

## 模块设计

遵循分层架构：`web.rs` 作为核心驱动，通过 from_wasm/to_wasm 消息系统与 JavaScript 桥接，子模块处理各自领域的平台操作。`web_socket` 为私有模块只暴露 `OsWebSocket` 结构体。
