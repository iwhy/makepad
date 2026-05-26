# Linux Direct 平台模块声明

## 概述

`mod.rs` 是 Linux Direct（DRM/GBM/EGL 直接渲染）平台子系统的模块根文件。该平台不依赖 X11 或 Wayland，直接在 Linux DRM 子系统中进行全屏渲染。

## 模块列表

| 模块名 | 用途 |
|--------|------|
| `direct_event` | 平台事件枚举（Paint、鼠标、键盘、文本输入、定时器） |
| `drm_sys` | Linux DRM (Direct Rendering Manager) FFI 绑定 |
| `egl_drm` | EGL + GBM + DRM 集成（窗口创建、上下文管理、翻页） |
| `gbm_sys` | Linux GBM (Generic Buffer Manager) FFI 绑定 |
| `linux_direct` | 主平台实现：事件循环、OpenGL 渲染、平台操作 |
| `raw_input` | `/dev/input/event*` 原始输入设备轮询 |

## 实现说明

- 所有模块通过 `pub mod` 公开。
- `linux_direct.rs` 是核心事件循环，驱动整个 Direct 平台。
- `raw_input.rs` 通过 Linux input 子系统直接读取键盘/鼠标/触摸事件。
- FFI 模块（`drm_sys`、`gbm_sys`）只声明 C 函数和类型，不含实现。
