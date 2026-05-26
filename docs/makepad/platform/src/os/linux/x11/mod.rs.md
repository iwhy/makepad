# mod.rs — X11 Module Declarations

**文件路径**: platform/src/os/linux/x11/mod.rs (7行)
**核心用途**: 声明 X11 后端的所有子模块。

## 子模块
| 模块名 | 用途 |
|--------|------|
| `linux_x11` | 主 X11 事件循环和 CxOsOp 实现 |
| `linux_x11_stdin` | Studio stdin-loop 模式的 X11 后端渲染 |
| `opengl_x11` | X11 窗口的 OpenGL/EGL 管理，DMA-BUF 导出 |
| `x11_sys` | X11 库 FFI 绑定和数据结构定义 |
| `xlib_app` | Xlib 应用层：事件循环、窗口管理、剪贴板、DnD |
| `xlib_event` | XlibEvent 枚举定义 |
| `xlib_window` | XlibWindow 结构体和窗口操作 |
