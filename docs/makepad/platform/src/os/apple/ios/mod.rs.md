# mod.rs — iOS 平台模块入口

**文件路径**: `platform/src/os/apple/ios/mod.rs` (6 行)
**核心作用**: 将 iOS 平台的所有子模块组织并公开到 crate 的 `os::apple::ios` 命名空间。

## 模块结构

| 模块 | 作用 |
|------|------|
| `ios` | 核心 iOS 平台实现：事件循环、渲染、摄像头、视频播放、权限 |
| `ios_app` | iOS 应用生命周期管理、UIKit 类注册、IME、系统交互 |
| `ios_delegates` | UIKit delegate/回调的 Objective-C 类定义 |
| `ios_event` | iOS 平台特定事件枚举 |
| `ios_text_input` | UITextInput 协议实现，支持 IME 输入 |

## 重新导出

- `pub use self::ios::*` — 将 `ios.rs` 中所有公开符号提升到 `ios` 模块顶层，外部通过 `crate::os::apple::ios::xxx` 直接访问。
