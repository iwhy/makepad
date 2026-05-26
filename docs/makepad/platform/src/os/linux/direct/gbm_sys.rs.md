# Linux GBM FFI 绑定

## 概述

`gbm_sys.rs` 是 Linux Generic Buffer Manager (GBM) API 的 FFI 绑定。GBM 是 Mesa 提供的缓冲区管理库，作为 EGL 和 DRM/KMS 之间的中间层。

通过 `#[link(name = "gbm")]` 链接 `libgbm`。

## 类型

| 类型 | 说明 |
|------|------|
| `gbm_device` | GBM 设备（不透明） |
| `gbm_surface` | GBM 表面（不透明） |
| `gbm_bo` | GBM 缓冲区对象（不透明） |
| `gbm_bo_handle` | 缓冲区句柄联合体（ptr/s32/u32/s64/u64） |

## 常量

- `GBM_FORMAT_XRGB8888 = 0x34325258`（"XR24"）— 32 位 XRGB 格式
- `GBM_BO_USE_SCANOUT = 1 << 0` — 可用于扫描输出
- `GBM_BO_USE_RENDERING = 1 << 2` — 可用于渲染

格式常量使用 `gbm_four_char_as_u32` 内置函数将 4 字符代码转换为 u32（"XR24" → `0x34325258`）。

## 外部函数

| 函数 | 说明 |
|------|------|
| `gbm_create_device` | 从 DRM fd 创建 GBM 设备 |
| `gbm_surface_create` | 创建 GBM 表面（指定宽/高/格式/标志） |
| `gbm_bo_get_stride` | 获取缓冲区 stride（行字节数） |
| `gbm_surface_lock_front_buffer` | 锁定前缓冲区用于渲染 |
| `gbm_surface_release_buffer` | 释放前缓冲区 |
| `gbm_bo_get_user_data` | 获取缓冲区用户数据 |
| `gbm_bo_get_handle` | 获取缓冲区句柄 |

## 实现说明

- GBM 是 DRM 和 EGL 之间的桥梁：`gbm_surface` 既可作为 EGL 窗口表面，也可从中提取 `gbm_bo` 用于 `drmModeAddFB2`。
- `gbm_bo_handle` 的联合体提供了多种访问方式，在 Makepad 中主要使用 `u32_` 成员传递给 `drmModeAddFB2`。
- 文件较小，只包含 GBM 的最小必要绑定（Makepad 未使用 `gbm_bo_map`、`gbm_bo_unmap` 等内存映射函数）。
