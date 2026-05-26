# File: mod.rs

- **核心用途**: Windows 平台模块的入口文件，通过 `pub mod` 声明导出所有子模块，是 `makepad_platform/src/os/windows/` 目录的模块根。
- **所属层级**: 平台层 · 模块集合根

---

## 模块声明

| 子模块 | 简介 |
|--------|------|
| `angle` | EGL/ANGLE OpenGL ES → D3D11 桥接层 |
| `common` | 公共类型、缩放因子、窗口状态与原生窗口包装 |
| `compositor` | DWM 合成器交互与帧同步 |
| `cpuid` | CPU 特性指令集检测 |
| `d3d11` | Direct3D 11 渲染管线完整封装 |
| `dcomp` | DirectComposition 合成 API 封装 |
| `dropsource` | OLE IDropSource 拖放源端实现 |
| `dxgi` | DXGI 适配器、输出和 Swapchain 管理 |
| `hwnd_control` | 窗口句柄样式控制与子类化 |
| `layer` | D3D11 渲染层抽象 |
| `mf_media_player` | Media Foundation 媒体播放器 |
| `mf_media_player_callback` | Media Foundation 异步事件回调 |
| `raw_handle` | 原生句柄的 Rust 安全包装 |
| `renderer` | Makepad 跨平台渲染器到 D3D11 的映射 |
| `runtime` | D3D11 运行时资源生命周期管理 |
| `swapchain` | Swapchain 帧缓冲管理 |
| `vs` | Windows 平台 Vert Shader 管理和 Shader 编译 |
| `win32_app` | Win32 应用生命周期（消息循环、启动） |
| `win32_clipboard` | 系统剪贴板读写封装 |
| `win32_window` | Win32 窗口创建、消息处理与事件映射 |
| `xaml_app` | UWP/XAML 应用宿主桥接 |

## 实现细节

- 使用 `#[allow(unused_variables)]` 属性忽略未使用变量的警告。
- 不重新导出或重新组织 API，仅作为模块集合的声明点。
- 所有子模块通过 `windows` 条件编译保护，仅在 `cfg(target_os = "windows")` 时编译。

## 平台集成

- 是 Makepad 跨平台架构中 Windows 后端的模块汇总。
- 上层代码通过 `crate::os::windows::xxx` 路径访问各模块功能。
