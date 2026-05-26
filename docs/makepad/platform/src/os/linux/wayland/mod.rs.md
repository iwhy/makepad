# mod.rs — Wayland Module Declarations

**文件路径**: platform/src/os/linux/wayland/mod.rs (6行)
**核心用途**: 声明 Wayland 后端的所有子模块。

## 子模块
| 模块名 | 用途 |
|--------|------|
| `linux_wayland` | 主 Wayland 事件循环和 CxOsOp 实现 |
| `opengl_wayland` | Wayland 窗口和弹出窗口的 OpenGL/EGL 表面管理 |
| `wayland_app` | 高层 Wayland 应用事件循环封装 |
| `wayland_state` | Wayland 协议状态管理、Dispatch trait 实现 |
| `wayland_type` | 鼠标光标和按钮的类型转换 |
| `xkb_sys` | XKB 键盘扩展的 FFI 绑定和安全封装层 |
