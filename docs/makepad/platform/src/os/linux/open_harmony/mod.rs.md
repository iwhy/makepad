# mod.rs

One-liner (EN): Module declarations for the OpenHarmony platform support subdirectory.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/open_harmony/mod.rs` (7 行)
- **核心作用**: 声明 `open_harmony/` 目录下的 7 个子模块，作为 OpenHarmony 平台支持的模块入口。

## 模块清单

| 模块 | 功能 |
|------|------|
| `arkts_obj_ref` | ArkTS 对象引用封装，提供跨语言（Rust↔ArkTS）属性读取与 JS 函数调用 |
| `oh_callbacks` | XComponent 和 VSync 原生回调注册，触摸事件、文本事件的分发 |
| `oh_media` | OpenHarmony 媒体 API 的桩实现（MIDI、音频、视频均为空操作） |
| `oh_sys` | FFI 绑定：OH_NativeVSync、libuv work queue、rawfile 资源管理 |
| `oh_util` | napi 工具函数：字符串/f64 提取、uv loop 获取、属性查询、globalThis/Context 获取 |
| `open_harmony` | 核心平台实现：EGL 窗口/上下文管理、事件循环、触摸/键盘/IME 处理 |
| `raw_file` | OpenHarmony 原生资源文件（Rawfile）的读取封装 |

## 平台集成

该模块通过 `pub mod` 导出，由上层 `platform/src/os/linux/mod.rs` 条件编译引入。仅在 `target_os = "ohos"` 时编译。
