# opengl_x11.rs — X11 OpenGL/EGL Window and Texture Management

**文件路径**: platform/src/os/linux/x11/opengl_x11.rs (500行)
**核心用途**: 管理 X11 窗口的 OpenGL/EGL 表面创建、窗口创建过程，以及 DMA-BUF 纹理导出/导入以支持 RunView swapchain。

## Types/Structs

### `OpenglWindow`
OpenGL 窗口组合体。

| 字段 | 类型 | 描述 |
|------|------|------|
| `first_draw` | `bool` | 首次绘制标志 |
| `window_id` | `WindowId` | 窗口 ID |
| `window_geom` | `WindowGeom` | 窗口几何信息 |
| `opening_repaint_count` | `u32` | 打开时的重绘计数 |
| `cal_size` | `Vec2d` | 物理像素尺寸缓存 |
| `xlib_window` | `Box<XlibWindow>` | 底层的 XlibWindow |
| `egl_surface` | `EGLSurface` | EGL 窗口表面 |

## Key Methods

### `OpenglWindow::new(window_id, opengl_cx, inner_size, position, title, is_fullscreen) -> OpenglWindow`
创建 X11 OpenGL 窗口：
1. 断言 EGL 平台为 `EGL_PLATFORM_X11_EXT`
2. 从 EGL 配置获取 `EGL_NATIVE_VISUAL_ID`
3. 通过 `XGetVisualInfo` 获取匹配的 X11 Visual
4. 创建 XlibWindow 并初始化
5. 创建 EGL 窗口表面（`eglCreateWindowSurface`）

### `OpenglWindow::new_popup(window_id, parent_window_id, opengl_cx, size, position) -> OpenglWindow`
创建弹出窗口：与 `new` 类似，但使用 `xlib_window.init_popup()`。

### `OpenglWindow::resize_buffers() -> bool`
检查 cal_size 是否变化，返回是否需要调整缓冲。

### `Cx::share_texture_for_presentable_image(texture) -> Option<LinuxOwnedImage>`
将 Makepad 纹理导出为 DMA-BUF（用于 Studio RunView swapchain）：
1. 确保纹理分配完成
2. 通过 `eglCreateImageKHR` 创建 EGL 图像
3. 通过 `eglExportDMABUFImageQueryMESA` 查询 fourcc/modifiers
4. 通过 `eglExportDMABUFImageMESA` 导出 DMA-BUF fd
5. 返回 `LinuxOwnedImage`（含 drm_format 和 plane）

### `Cx::upload_presentable_image_software_buffer(texture, width, height, pixels)`
软件回退路径：将像素数据上传到现有 GL 纹理（`glTexSubImage2D`）。

### `CxTexture::update_shared_texture(gl)`
初始化共享纹理的 GL 存储：生成纹理对象，设置 NEAREST 过滤，`glTexImage2D` 分配。

### `CxTexture::update_from_shared_dma_buf_image(gl, opengl_cx, dma_buf_image)`
从 DMA-BUF 导入纹理：
1. 通过 `eglCreateImageKHR` + `EGL_LINUX_DMA_BUF_EXT` 创建 EGL 图像
2. 通过 `glEGLImageTargetTexture2DOES` 绑定到 GL 纹理
