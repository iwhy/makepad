# File: angle.rs

- **核心用途**: 封装 EGL/ANGLE 初始化逻辑，将 OpenGL ES 渲染能力桥接到 Direct3D 11 后端，使 Makepad 能够在 Windows 上通过 D3D11 设备运行 OpenGL ES 渲染管线。
- **所属层级**: 平台层 · Windows 操作系统抽象
- **涉及概念**: EGL, ANGLE, OpenGL ES, D3D11, display, surface, context, config

---

## 类型与结构体

| 名称 | 可见性 | 简介 |
|------|--------|------|
| `Angle` | `pub` | EGL/ANGLE 状态容器，包含 display、surface、context、config 等所有 EGL 对象的生命周期，以及 D3D11 设备引用和 libEGL/libGLESv2 的动态链接库句柄。 |
| `Gles` | `pub` | 封装单个 OpenGL ES 渲染实例，包含 fbo、color buffer、depth stencil、multisample 等 render target 资源。 |

## 核心方法

### `Angle`

| 方法 | 可见性 | 简述 |
|------|--------|------|
| `new_for_d3d11(device: &Device, native: &Native) -> Option<Self>` | `pub` | 从已有的 D3D11 设备和原生窗口句柄创建 ANGLE 实例。依次：加载 libEGL/libGLESv2 → 获取 display → 初始化 → 选择 config → 创建 context → 创建 surface → 设置 current context。 |
| `new_for_window(device: &Device) -> Option<Self>` | `pub` | 通过 `CreateWindowA` 创建一个隐藏的辅助窗口，然后调用 `new_for_d3d11`。 |
| `make_current(&self, surface: EGLSurface)` | `pub` | 将指定 surface 设为当前渲染上下文和绘图表面。 |
| `destroy(&mut self)` | `pub` | 安全析构：释放 surface、context、display，并卸载动态链接库。 |

### `Gles`

| 方法 | 可见性 | 简述 |
|------|--------|------|
| `new(egl: &Angle, size: usize, size2: usize, format: u32) -> Option<Self>` | `pub` | 创建 GLES 离屏渲染目标：生成 fbo、color texture、depth stencil texture，可选 multisample resolve buffer。 |
| `make_current(&self, angle: &Angle)` | `pub` | 将此 GLES 实例的 fbo 设为当前帧缓冲。 |
| `destroy(&mut self, egl: &Angle)` | `pub` | 删除所有 GL 资源对象。 |

## 实现细节

- 使用 `libloading` crate 动态加载 `libEGL.dll` 和 `libGLESv2.dll`，所有 EGL 函数通过函数指针调用。
- config 属性通过 `EGL_CONFIG_ATTRIBUTES` 硬编码数组指定：要求 EGL 1.3 以上、窗口 surface、BGRA8、depth=24、stencil=8。
- `EGL_EXT_platform_base` 和 `EGL_ANGLE_platform_angle_d3d11` 扩展用于非 Windows 原生平台（Windows 上直接用 eglCreateWindowSurface）。
- 使用 `EGL_ANGLE_d3d11_share_device` 扩展将外部 D3D11 设备导入 ANGLE。
- Gles 的 fbo 通过 `glFramebufferTexture2D` 绑定，格式根据 `format` 参数选择 `GL_RGBA8` 或 `GL_R32F`。
- 支持 `GL_EXT_multisampled_render_to_texture` 或手动 resolve 实现 MSAA。

## 平台集成

- **Windows ANGLE DLL 依赖**：需要系统中存在 libEGL.dll 和 libGLESv2.dll（通常由 ANGLE 项目或 GPU 驱动提供 D3D11 转发层）。
- **D3D11 共享设备**：ANGLE 复用外部 D3D11 设备，确保 GPU 资源在同一设备上下文中共享。
- **辅助窗口**：`new_for_window` 创建的隐藏窗口仅用于 EGL 初始化阶段，实际渲染 surface 来自外部传入的原生窗口句柄。
