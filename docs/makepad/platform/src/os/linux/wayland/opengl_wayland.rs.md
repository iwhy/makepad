# opengl_wayland.rs — Wayland OpenGL/EGL Window Management

**文件路径**: platform/src/os/linux/wayland/opengl_wayland.rs (414行)
**核心用途**: 管理 Wayland 主窗口和弹出窗口的 EGL 表面、xdg_shell 协议交互、窗口图标、缓冲缩放。

## Types/Structs

### `WaylandWindow`
Wayland 主窗口。

| 字段 | 类型 | 描述 |
|------|------|------|
| `window_id` | `WindowId` | 窗口唯一标识 |
| `base_surface` | `WlSurface` | 基础的 wl_surface |
| `toplevel` | `XdgToplevel` | xdg_toplevel 角色对象 |
| `decoration` | `Option<ZxdgToplevelDecorationV1>` | 窗口装饰对象（CSD） |
| `xdg_surface` | `XdgSurface` | xdg_surface 对象 |
| `viewport` | `Option<WpViewport>` | viewport 缩放（若合成器支持） |
| `fractional_scale` | `Option<WpFractionalScaleV1>` | 分數縮放（若合成器支持） |
| `configured` | `bool` | 是否已收到首次 configure |
| `window_geom` | `WindowGeom` | 窗口几何信息 |
| `cal_size` | `Vec2d` | 上次计算的物理像素尺寸 |
| `wl_egl_surface` | `WlEglSurface` | Wayland EGL 表面包装 |
| `egl_surface` | `EGLSurface` | EGL 窗口表面 |

### `WaylandPopupWindow`
Wayland 弹出窗口。

| 字段 | 类型 | 描述 |
|------|------|------|
| `window_id` | `WindowId` | 窗口 ID |
| `parent_window_id` | `WindowId` | 父窗口 ID |
| `base_surface` | `WlSurface` | wl_surface |
| `xdg_surface` | `XdgSurface` | xdg_surface |
| `xdg_popup` | `XdgPopup` | xdg_popup 角色 |
| `viewport` | `Option<WpViewport>` | viewport 缩放 |
| `fractional_scale` | `Option<WpFractionalScaleV1>` | 分數縮放 |
| `wl_egl_surface` | `Option<WlEglSurface>` | Wayland EGL 表面 |
| `egl_surface` | `EGLSurface` | EGL 窗口表面 |
| `egl_display` | `EGLDisplay` | EGL 显示 |
| `egl_destroy_surface_fn` | 函数指针 | 销毁 EGL 表面的 libegl 函数 |
| `window_geom` | `WindowGeom` | 窗口几何 |
| `configured` | `bool` | 是否已配置 |
| `cal_size` | `Vec2d` | 物理像素尺寸缓存 |

## Key Methods

### `WaylandWindow::new(...) -> WaylandWindow`
创建 Wayland 主窗口：
1. 创建 `wl_surface` 和 `xdg_surface`/`xdg_toplevel`
2. 设置标题、app_id、窗口图标（通过 xdg_toplevel_icon_v1）
3. 请求客户端侧窗口装饰（SSD→CSD）
4. 若全屏则调用 `set_fullscreen`
5. 创建 `WlEglSurface` 和 EGL 窗口表面
6. 初始化 `WindowGeom`

### `WaylandWindow::set_wayland_icon(icon_manager, shm, toplevel, qhandle)`
通过 xdg-toplevel-icon-v1 协议设置窗口图标：
1. 获取应用图标 RGBA8 数据
2. 转换 RGBA8 为 ARGB8888 字节序
3. 通过 memfd_create + mmap 创建共享内存缓冲
4. 创建 wl_shm_pool + wl_buffer 
5. 创建 xdg_toplevel_icon 并调用 `set_icon`

### `WaylandWindow::resize_buffers() -> bool`
检查窗口逻辑尺寸 × DPI 是否变化，若有变化则调用 `wl_egl_surface.resize`。

### `WaylandWindow::close_window()`
按协议顺序销毁 Wayland 对象：decoration → toplevel → xdg_surface → viewport → fractional_scale → base_surface。

### `WaylandPopupWindow::new(...) -> WaylandPopupWindow`
创建弹出窗口：
1. 创建 wl_surface 和 xdg_surface
2. 配置 xdg_positioner（尺寸、锚点矩形、锚点/重力、约束调整）
3. 创建 xdg_popup（不 grab，让应用完全控制弹出窗口生命周期）
4. 创建 EGL 窗口表面
5. 初始化 WindowGeom

### `WaylandPopupWindow::resize_buffers() -> bool`
同 WaylandWindow，检查并调整 EGL 缓冲尺寸。

### `WaylandPopupWindow::close_window()`
按正确顺序销毁：EGL 表面 → wl_egl_surface → xdg_popup → xdg_surface → viewport → fractional_scale → base_surface。

## 析构实现
`WaylandWindow` 和 `WaylandPopupWindow` 的 `Drop` 实现自动调用 `close_window()`。

## Implementation Details
- EGL 平台必须是 `EGL_PLATFORM_WAYLAND_KHR`（断言检查）
- 弹出窗口不进行 grab，避免合成器自动关闭
- constaint_adjustment 设置所有方向（Flip/Slide X/Y），确保弹出窗口在屏幕边界内
- `cal_size` 用于避免不必要的缓冲调整
