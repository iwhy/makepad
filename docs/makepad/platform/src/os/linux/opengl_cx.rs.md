# opengl_cx.rs — EGL 上下文创建与帧缓冲渲染

**文件路径**: `platform/src/os/linux/opengl_cx.rs` (438 行)
**核心功能**: EGL 显示初始化、OpenGL 上下文创建、以及将绘制结果渲染到 EGL surface 并交换缓冲区的实现。

## 主要类型

### `OpenglCx`
OpenGL 上下文状态：
- `libegl` / `libgl` — EGL 和 OpenGL 函数表
- `egl_display` / `egl_config` / `egl_context` — EGL 核心对象
- `egl_platform` / `egl_platform_display` — 平台类型和原生显示指针

## 关键方法

### `OpenglCx::from_egl_platform_display(egl_platform, egl_platform_display)`
从平台显示创建 EGL 上下文：
1. 通过 `LibEgl::try_load()` 加载 EGL 函数表
2. 使用 `eglGetPlatformDisplayEXT` 获取 EGL 显示
3. 初始化 EGL 并绑定 OpenGL ES API
4. 从降级链中选择帧缓冲配置（优先 RGBA8888 + Depth24 + Stencil8 + ES3）：

| 优先级 | 配置 |
|--------|------|
| 1 | RGBA8 + Depth24 + Stencil8 + Window + ES3 |
| 2 | RGBA8 + Depth24 + Window + ES3 |
| 3 | RGBA8 + Depth16 + Window + ES3 |
| 4 | RGBA8 + Window + ES3（旧行为回退） |

5. 创建 OpenGL ES 3.0 上下文
6. 通过 `LibGl::try_load` 使用 `eglGetProcAddress` 加载 OpenGL 函数

### `OpenglCx::make_current()`
将 EGL 上下文设为当前（无 surface 绑定）。

### `Cx::draw_pass_to_window(draw_pass_id, egl_surface, pix_width, pix_height)`
将绘制通道渲染到 EGL window surface：
1. 调用 `eglMakeCurrent` 绑定 surface
2. 设置 viewport
3. 配置绘制通道（clear color/depth）
4. 调用 `Cx::render_view` 执行实际渲染
5. 处理可选调试功能：
   - `MAKEPAD_GL_READBACK` — 中心像素读取
   - `MAKEPAD_WRITE_FRAMEBUFFER_PNG` — 帧缓冲写入 PNG
   - Studio 截图（读取帧缓冲，行翻转，编码为 PNG）
6. 调用 `eglSwapBuffers` 交换缓冲区
7. 处理 swap 错误并记录日志

### `egl_error_name(error)`
将 EGL 错误码转为人类可读的字符串。
