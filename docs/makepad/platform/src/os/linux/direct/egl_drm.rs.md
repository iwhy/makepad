# EGL + DRM + GBM 渲染实现

## 概述

`egl_drm.rs` 实现了 EGL + GBM + DRM/KMS 的全屏渲染栈，用于 Linux Direct 平台。它负责：
1. 通过 DRM 发现显示设备并选择最佳模式
2. 使用 GBM 创建缓冲区管理器
3. 初始化 EGL 上下文和窗口表面
4. 管理翻页和垂直同步

## 核心类型

### `Drm`
DRM/KMS 状态管理：
- `width` / `height` — 显示分辨率
- `bo_fb_ids` — GBM 缓冲区到 DRM FB ID 的缓存映射
- `fourcc_format` — 像素格式（XRGB8888）
- `drm_fd` — DRM 文件描述符
- `drm_mode` / `drm_resources` / `drm_connector` / `drm_encoder` — KMS 资源
- `gbm_dev` / `gbm_surface` — GBM 设备和表面
- `current_bo` — 当前显示中的缓冲区

### `Egl`
EGL 状态管理：
- `libegl` — EGL 库句柄
- `egl_display` — EGL 显示连接
- `egl_surface` — EGL 窗口表面
- `egl_context` — EGL 渲染上下文

## `Drm` 关键方法

### `new(mode_want: &str) -> Option<Self>`
初始化 DRM/GBM：
1. 调用 `drmGetDevices2` 枚举 DRM 设备
2. 遍历设备，打开每个设备的 `DRM_NODE_PRIMARY` 节点
3. 遍历连接器，找到第一个 `DRM_MODE_CONNECTED` 的连接器
4. 匹配请求的显示模式（格式如 `"1280x720-60"`）
5. 找到对应的编码器
6. 创建 GBM 设备和表面（`XRGB8888` + `SCANOUT | RENDERING` 标志）

### `get_fb_id_for_bo(bo) -> u32`
获取/创建 GBM 缓冲区对应的 DRM FB ID：
- 先在缓存中查找，找到则复用
- 未找到时调用 `drmModeAddFB2` 注册新帧缓冲区

### `first_mode()`
渲染前设置 CRTC：锁定前缓冲区 → 获取 FB ID → `drmModeSetCrtc`。

### `swap_buffers_and_wait(egl)`
完成一帧渲染：
1. EGL `swap_buffers`（渲染到 GBM）
2. 锁定新的前缓冲区
3. `drmModePageFlip` 请求翻页（带 `PAGE_FLIP_EVENT`）
4. 使用 `select` + `drmHandleEvent` 等待翻页完成回调
5. 释放旧缓冲区

翻页完成回调（`handle_page_flip`）将等待标志置零，解除 `while` 循环。

## `Egl` 关键方法

### `new(drm: &Drm) -> Option<Self>`
初始化 EGL：
1. 通过 `eglGetPlatformDisplayEXT` 获取 GBM 平台显示
2. `eglInitialize` 初始化 EGL
3. `eglBindAPI(EGL_OPENGL_ES_API)` 绑定 OpenGL ES
4. 选择匹配 XRGB8888 Native Visual ID 的 EGL 配置
5. 创建 EGL 上下文（OpenGL ES 2.0）
6. 创建 EGL 窗口表面（使用 `gbm_surface`）
7. 调用 `eglMakeCurrent`
8. 通过 `eglGetProcAddress` 加载 OpenGL 函数

### `make_current()`
重新绑定 EGL 上下文和表面。

### `swap_buffers()`
调用 `eglSwapBuffers` 交换前后缓冲区。

## 实现说明

- `mode_want` 格式为 `"<width>x<height>-<refresh>"`，如 `"3840x2160-60"`。
- 翻页使用阻塞式 `select` 等待，不支持超时或中断。
- EGL 配置选择硬编码：RGB 各 1 位、无 Alpha、无深度、OpenGL ES 2。
- 使用 `Drm` 的 `bo_fb_ids` 缓存避免重复调用 `drmModeAddFB2`。
- DRM 设备发现中忽略没有主节点的设备（`available_nodes & (1 << DRM_NODE_PRIMARY) == 0`）。
