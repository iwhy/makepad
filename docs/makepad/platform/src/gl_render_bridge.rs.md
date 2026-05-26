# `gl_render_bridge.rs` — GL 渲染桥接器（跨平台共享 GL 上下文）

## 文件定位

该文件实现了 Makepad 框架中**最重要的 GPU 互操作性组件**——`GlRenderBridge`。它允许外部的 OpenGL 渲染代码在 Makepad 管理的 GL 上下文（或与 Makepad 共享 GPU 内存的上下文中）运行，并将渲染结果零拷贝地在 Makepad 原生渲染管线中显示。这是嵌入第三方 GL 渲染器（如 Servo/WebRender、Google Maps GL、自定义游戏引擎）的核心接口。

`GlRenderBridge` 在每个平台上有完全不同的底层实现，但对外暴露统一的 API。平台矩阵：

| 平台 | 底层实现 | 共享机制 |
|---|---|---|
| **Linux** | `EglRenderBridge` (EGL) | 共享 Makepad 的 EGL 上下文 |
| **Android** | `EglRenderBridge` (EGL) | 共享 Makepad 的 EGL 上下文 |
| **Windows** | `AngleRenderBridge` (ANGLE) | 在 D3D11 设备上创建 ANGLE EGL 上下文 |
| **macOS** | `CglRenderBridge` (CGL) | 独立 CGL 上下文 + IOSurface 桥接到 Metal |
| **iOS** | `EaglRenderBridge` (EAGL) | 独立 EAGL 上下文 + IOSurface/CVPixelBuffer 桥接到 Metal |

---

## `GlApi` 枚举

```rust
pub enum GlApi {
    GL,   // 桌面 OpenGL (macOS)
    GLES, // OpenGL ES (Linux, Android, Windows via ANGLE)
}
```

区分外部渲染代码使用的 GL API 方言。macOS 原生是桌面 OpenGL，其他 EGL 平台使用 OpenGL ES。外部渲染器在调用 `get_proc_address` 获取函数指针时应使用对应的 API 版本。

---

## `GlRenderBridge` 结构体

```rust
pub struct GlRenderBridge {
    // 平台特定的内部实现，条件编译选择
    inner: EglRenderBridge | AngleRenderBridge | CglRenderBridge | EaglRenderBridge,
}
```

每个变体都是一个完整的渲染上下文管理结构。各平台保证该结构体在生产环境下有实际功能，在编译时确认目标平台不会产生悬挂类型。

### 跨平台通用方法

#### `fn make_current(&self)`
将桥接器的 GL 上下文绑定到当前线程。外部渲染器在发起任何 GL 调用之前必须首先调用此方法。内部实现：
- **Linux/Android**: 调用 `eglMakeCurrent` 将桥接器的 EGL 上下文绑定到线程。
- **Windows**: ANGLE 的 `eglMakeCurrent` 等效。
- **macOS**: `CGLSetCurrentContext` 设置 CGL 上下文。
- **iOS**: `EAGLContext setCurrent:` 设置 EAGL 上下文。

#### `fn get_proc_address(&self, name: &str) -> *const c_void`
按名称查找 GL/EGL 函数指针。外部渲染器使用此方法动态加载 GL 函数，避免静态链接平台特定的 GL 库实现。返回的函数指针可转换为具体的 GL 函数类型。

#### `fn gl_api(&self) -> GlApi`
查询当前桥接器的 GL API 类型。外部渲染器根据返回值决定调用桌面 OpenGL 还是 OpenGL ES 的函数。

### EGL 平台特定方法（Linux, Android, Windows）

```rust
fn egl_display(&self) -> *mut c_void;
fn egl_config(&self) -> *mut c_void;
fn egl_context(&self) -> *mut c_void;
```

暴露底层的 EGL 对象句柄，供高级用户直接操作 EGL 对象。典型用途：在第三方库（如 FFmpeg 的硬件加速解码路径）需要直接使用 EGL 显示和上下文时提供。这些方法只在 EGL 基础平台上可用，在 macOS/iOS 上不编译。

### CGL 平台特定方法（macOS）

```rust
fn cgl_pixel_format(&self) -> *mut c_void;
fn cgl_context(&self) -> *mut c_void;
```

暴露 macOS 的 CGL 像素格式和上下文句柄。用于需要直接操纵 CGL 的第三方渲染库（如 `CGLTexImageIOSurface2D` 等）。

### EAGL 平台特定方法（iOS）

```rust
fn eagl_context(&self) -> *mut c_void;
fn opengles_framework(&self) -> *mut c_void;
```

暴露 iOS 的 EAGL 上下文句柄和 OpenGLES framework 指针。

---

## 各平台 `Cx` 方法

### `create_gl_render_bridge(&mut self) -> GlRenderBridge`

根据平台创建对应类型的桥接器：

- **Linux**: 从 `self.os.opengl_cx` 获取现有 EGL 显示、配置、上下文，以及 `eglGetProcAddress` 和 `eglMakeCurrent` 函数指针，构造 `EglRenderBridge`。
- **Android**: 从 `self.os.display` 获取现有 EGL 对象，构造方式与 Linux 相同。
- **macOS**: 创建独立的 CGL 上下文（GL 3.2 Core Profile），不依赖 Makepad 的 Metal 上下文。
- **Windows**: 获取 Makepad 的 D3D11 设备，在其上创建 ANGLE EGL 上下文。
- **iOS**: 创建独立的 EAGL 上下文（GLES 3.0），与 Metal 上下文分离。

macOS 和 iOS 上创建的是**独立上下文**，意味着外部渲染可以在独立线程中执行而不干扰 Makepad 的主渲染线程。Linux/Android 上是**共享上下文**，必须在线程安全的前提下使用。

### `create_gl_render_bridge_texture(&mut self, bridge, width, height) -> (Texture, u32)`

创建一张可被外部 GL 渲染写入、被 Makepad 原生渲染器显示的纹理：

- **Linux/Android**: 委托给 `self.create_gl_render_texture(width, height)`。此方法创建一个 Makepad 管理的纹理，并返回其 GL 纹理 ID。外部渲染器直接渲染到该 GL 纹理上，Makepad 通过共享 EGL 上下文直接读取渲染结果。

- **macOS**: 通过 IOSurface 实现零拷贝共享：
  1. 调用 `bridge.inner.make_current()` 切换到 CGL 上下文。
  2. 调用 `self.create_iosurface_render_texture` 创建 IOSurface 后端纹理，得到 Makepad `Texture`、`IOSurfaceRef` 和 ID。
  3. 调用 `bridge.inner.bind_iosurface_to_gl_texture` 将同一个 IOSurface 绑定到 CGL 上下文的 GL 纹理上。
  4. 返回 Makepad Metal 纹理句柄和 CGL GL 纹理 ID。Metal 和 GL 通过 IOSurface 共享内存，无需 CPU 拷贝。

- **Windows**: 委托给 `bridge.inner.create_render_texture(self, width, height)`。ANGLE 后端在 D3D11 共享资源上创建 GL 纹理，Makepad 的 D3D11 渲染器直接共享该资源。

- **iOS**: 使用 CVPixelBuffer 作为共享后端，避免手动创建 IOSurface 后的 `-6683` 错误：
  1. `bridge.inner.make_current()` 切换到 EAGL 上下文。
  2. 获取 Metal device。
  3. 调用 `bridge.inner.create_shared_texture(metal_device, width, height)` 同时创建 GLES 纹理和 Metal 纹理（通过 CVMetalTextureCache）。
  4. 创建 `SharedBGRAu8` 格式的 Makepad `Texture`，注入 Metal 纹理引用。
  5. 返回 Makepad 纹理句柄和 GLES 纹理 ID。

### `restore_gl_context(&mut self)`

恢复 Makepad 原生渲染器的 GL 上下文状态：
- **Linux/Android**: 由于桥接器共享 EGL 上下文，外部渲染器可能修改了 GL 状态（绑定了 VAO、VBO、FBO、纹理、着色器等）。此方法显式将所有绑定重置为默认值（0），禁用剪裁测试、混合，恢复颜色掩码和深度掩码，以确保 Makepad 渲染器从干净状态开始。
- **macOS/iOS/Windows**: No-op。因为这些平台使用独立上下文，外部渲染不会污染 Makepad 的上下文状态。各上下文完全隔离。

---

## 条件编译的空实现

```rust
#[cfg(not(any(platforms_with_gl_support)))]
impl GlRenderBridge {
    pub fn make_current(&self) {}
    pub fn get_proc_address(&self, _name: &str) -> *const c_void { std::ptr::null() }
    pub fn gl_api(&self) -> GlApi { GlApi::GLES }
}
```

在无 GL 后端的平台（如 `wasm32`、`tvos` 的 headless 模式）上提供空实现。所有方法在编译时保留（避免条件编译宏污染调用者），但运行时无实际效果。

---

## 设计要点

1. **共享 vs 独立上下文模型**: Linux/Android 使用共享上下文（零拷贝，但有状态污染风险），macOS/iOS/Windows 使用独立上下文（状态隔离，但需 IOSurface/D3D11 共享资源桥接）。两种模型由 `restore_gl_context` 方法统一了安全退出路径。
2. **IOSurface 零拷贝 (macOS/iOS)**: Apple 平台上使用 IOSurface（macOS）或内部使用 IOSurface 的 CVPixelBuffer（iOS）实现 Metal 与 GL 之间的零拷贝共享，避免了昂贵的纹理拷贝。这是 Apple 生态系统中推荐的 GPU 互操作方式。
3. **ANGLE 桥接 (Windows)**: 在 Windows 上通过 ANGLE 将 GLES 调用转换为 D3D11，共享 D3D11 资源实现零拷贝。这种方式让 Windows 平台无需安装原生 GL 驱动即可运行 GL 渲染。
4. **状态安全**: 共享上下文平台的 `restore_gl_context` 是一个防御性的状态重置，可防止外部渲染器遗留的脏状态导致 Makepad 渲染出现难以调试的 artifacts。重置内容包括 VAO、VBO、FBO、纹理、着色器程序、混合/裁剪/颜色掩码等所有常见 GL 状态。
5. **纹理双重句柄**: `create_gl_render_bridge_texture` 返回 `(Texture, u32)`，即 Makepad 纹理句柄 + 原生 GL 纹理 ID。外部代码使用 GL 纹理 ID 渲染，Makepad 使用其纹理句柄显示，两者共享同一 GPU 内存。
