# egl_sys.rs — EGL FFI 绑定与上下文创建

**文件路径**: `platform/src/os/linux/egl_sys.rs` (665 行)
**核心功能**: EGL 库的 FFI 绑定，包括类型定义、函数指针结构体和跨平台的 EGL 上下文创建辅助函数。

## EGL 类型定义

### 基础类型
- `EGLDisplay` / `EGLConfig` / `EGLSurface` / `EGLContext` — 核心 EGL 对象类型
- `EGLNativeDisplayType` / `EGLNativeWindowType` — 原生平台类型
- `EGLint` / `EGLenum` / `EGLBoolean` — C 整数类型映射

### 关键常量
- `EGL_OPENGL_ES2_BIT` / `EGL_OPENGL_ES3_BIT_KHR` — ES 版本位标记
- `EGL_PLATFORM_X11_EXT` / `EGL_PLATFORM_WAYLAND_KHR` / `EGL_PLATFORM_GBM_KHR` — 平台枚举
- `EGL_LINUX_DMA_BUF_EXT` / `EGL_DMA_BUF_PLANE0_*` — DMA-BUF 导入属性
- `EGL_CONTEXT_MAJOR_VERSION` / `EGL_CONTEXT_MINOR_VERSION_KHR`

## `LibEgl` 结构体

通过 `dlopen` 动态加载 `libEGL.so`，包含以下函数指针分组：

### 核心 EGL 函数
`eglGetDisplay` / `eglInitialize` / `eglTerminate` / `eglChooseConfig` / `eglCreateContext` / `eglMakeCurrent` / `eglSwapBuffers` / `eglGetError` / `eglGetProcAddress` / `eglQueryString` 等。

### Surface 相关
`eglCreateWindowSurface` / `eglCreatePbufferSurface` / `eglCreatePixmapSurface` / `eglDestroySurface` / `eglQuerySurface` / `eglSurfaceAttrib` / `eglSwapInterval` / `eglBindTexImage` / `eglReleaseTexImage`。

### 扩展函数（通过 `eglGetProcAddress` 按名称加载）
- `eglCreateImageKHR` / `eglDestroyImageKHR` — EGLImage 创建
- `eglExportDMABUFImageQueryMESA` / `eglExportDMABUFImageMESA` — DMA-BUF 导出
- `eglGetPlatformDisplayEXT` — 平台显示获取
- `glEGLImageTargetTexture2DOES` — OpenGL 扩展（通过 EGL 加载）

## 跨平台上下文创建

### `create_egl_context` (Android)
为 Android 平台创建 EGL 上下文：
- 使用 `eglGetDisplay` 获取默认显示
- 配置 RGBA8888 + Depth24 的窗口 surface 配置
- 创建 OpenGL ES 3.0 上下文
- 自动回退到第一个可用配置

### `create_egl_context_openxr` (Android)
为 Android XR 创建 EGL 上下文，配置略有不同（8位 alpha、使用 `eglGetConfigs` 枚举所有可用配置、要求 `EGL_WINDOW_BIT | EGL_PBUFFER_BIT` 和 `EGL_OPENGL_ES3_BIT_KHR`）。

### `create_egl_context` (OHOS)
为 OpenHarmony 创建 EGL 上下文，模拟器和真机使用不同的 EGL 属性集。

## `LibEgl::try_load()`
动态加载 EGL 库的顺序：优先 `libEGL.so`，回退到 `libEGL.so.1`。
