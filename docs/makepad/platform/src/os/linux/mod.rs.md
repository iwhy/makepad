# mod.rs — Linux 平台模块声明

**文件路径**: `platform/src/os/linux/mod.rs` (100 行)
**核心功能**: Linux 平台所有模块的条件编译声明和公共类型重导出。

## 模块声明

### 桌面 Linux（非 direct、非 OHOS、非 Android）
- `opengl_cx` — EGL 上下文
- `wayland` — Wayland 窗口后端
- `windowing_backend` — 窗口系统后端选择
- `x11` — X11 窗口后端

### Linux Direct 模式
- `direct` — 直接渲染模式

### Android
- `openxr` / `openxr_anchor` / `openxr_depth` / `openxr_input` / `openxr_sys` — OpenXR 相关

### OpenHarmony
- `open_harmony` — OHOS 平台支持

### 通用模块（所有 Linux 变体）
- `egl_sys` / `gl_sys` / `gl_video_upload` / `libc_sys` / `module_loader` / `opengl`
- `vulkan` / `vulkan_naga`（条件编译：`use_vulkan`）

### 桌面独占（非 OHOS、非 Android）
- `dma_buf` / `gstreamer_sys` / `ipc` / `linux_video_playback` / `linux_video_player`
- `alsa_audio` / `alsa_midi` / `alsa_sys` / `linux_media`
- `v4l2_camera` / `v4l2_camera_player` / `v4l2_sys`
- `pulse_audio` / `pulse_sys`
- `socket_stream`

### 非 Android
- `select_timer`

## 类型重导出

- `CxOs` — 桌面 Linux 从 `self::windowing_backend::*` 导出
- `CxOs` — Android 从 `self::android::android::CxOs` 导出
- `CxOs` — OHOS 从 `self::open_harmony::open_harmony::*` 导出
- `CxOs` — Direct 从 `self::direct::linux_direct::*` 导出
- `OsMidiInput` / `OsMidiOutput` — 桌面 Linux 从 `alsa_midi`，Android 从 `android_midi`
