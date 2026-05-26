# `lib.rs` — platform crate 根与公共 API 导出

`lib.rs` 是 Makepad `platform` crate 的根文件，主要职责是声明所有模块并为 crate 外部使用者集中导出完整的公共 API。

---

## 模块声明

### 公开模块（`pub mod`）

| 模块 | 用途 |
|------|------|
| `action` | 动作系统（`ActionTrait`、`Actions`、类型安全向下转型） |
| `app_main` | `app_main!` 宏定义和 `AppMain` trait |
| `audio` | 音频输入/输出 |
| `audio_stream` | 音频流处理 |
| `component` | `ComponentRegistry` — 组件注册表 |
| `display_context` | 显示上下文和系统栏外观 |
| `event` | 事件系统（键盘、鼠标、触控、窗口、XR、文本等事件类型） |
| `file_dialogs` | 文件对话框 |
| `game_input` | 游戏手柄输入 |
| `ime` | 输入法编辑器（软键盘配置、自动修正等） |
| `midi` | MIDI 输入 |
| `os` | 操作系统抽象层 |
| `permission` | 权限管理 |
| `script` | Makepad 脚本运行时（`ScriptVm`、脚本对象系统） |
| `thread` | 线程工具 |
| `ui_runner` | `UiRunner<T>` 跨线程 UI 执行器 |
| `video` | 视频相关 |
| `web_socket` | WebSocket 客户端 |

### 私有模块（`mod` 无 `pub`）

| 模块 | 用途 |
|------|------|
| `cx` | 核心 `Cx` 上下文（窗口、绘制、事件循环管理器） |
| `cx_api` | Cx OS API trait 和平台操作枚举 |
| `arc_string_mut` | 可变字符串的原子引用计数包装 |
| `shared_bytes` | 跨线程共享字节缓冲区 |
| `draw_list` | 绘制列表 |
| `draw_matrix` | 绘制矩阵变换 |
| `draw_pass` | 绘制通道（render pass） |
| `draw_shader` | 绘制着色器 |
| `draw_vars` | 绘制变量 |
| `area` | 区域（Area）系统 |
| `component_list` | 组件列表 |
| `component_map` | 组件映射 |
| `cursor` | 鼠标光标 |
| `debug` | 调试工具 |
| `geometry` | 几何体 |
| `gpu_info` | GPU 信息 |
| `id_pool` | 通用 ID 池（Generation-based） |
| `live_reload` | 热重载观察者 |
| `macos_menu` | macOS 菜单栏 |
| `performance_stats` | 性能统计 |
| `texture` | 纹理管理 |
| `uniform_buffer` | Uniform 缓冲 |
| `window` | 窗口管理（`WindowHandle`、`CxWindow`、DPI 转换） |
| `xr_tsdf` | XR 截断有符号距离场 |
| `media_api` | 媒体 API |
| `media_host` | 媒体宿主 |
| `media_plugin` | 媒体插件 |
| `playback_session` | 播放会话 |
| `video_session` | 视频会话 |

### 条件编译模块

- **`#[cfg(not(target_arch = "wasm32"))]`**：`video_decode`（视频解码）、`video_encode`（视频编码）
- **`#[cfg(any(target_os = "macos", target_os = "windows", target_os = "linux"))]`**：`app_icon`（应用图标）
- **`#[macro_use]`**：`log`（日志宏）、`cx`（Cx 内部宏）

---

## 公共重新导出

此 crate 使用两个层次的导出模式：

### 1. 外部 crate 重导出

```rust
pub use makepad_futures;
pub use makepad_script_std::makepad_network;
pub use makepad_script_std::makepad_script;
pub use makepad_studio_protocol as studio;
```

- **`makepad_futures`**：异步运行时的 future 类型。
- **`makepad_network`**：HTTP 请求/响应、网络错误等网络工具。
- **`makepad_script`**：Makepad 脚本语言的所有核心类型（`ScriptValue`、`ScriptVm`、`Apply` 等）。
- **`makepad_studio_protocol`**：Studio 远程协议类型。
- **`makepad_script::trap`**：重新导出 `trap` 模块，供 `Script` derive 宏的 `crate::trap::ScriptTrap` 引用使用。

### 2. 内部模块批量导出

```rust
pub use { crate:: { ... }, app_main::*, ... };
```

将以下模块的核心类型提升到 crate 根级别：

- **`action`**：`Action`、`Actions`、`ActionsBuf`、`ActionCast`、`ActionCastRef`、`ActionDefaultRef`、`ActionTrait`
- **`area`**：`Area`、`InstanceArea`、`RectArea`
- **`component`**：`ComponentInfo`、`ComponentRegistries`、`ComponentRegistry`
- **`cx`**：`Cx`、`CxRef`、`LinuxWindowParams`、`OsType`
- **`cx_api`**：`AccessibilityUpdatePayload`、`CxOsApi`、`CxOsOp`、`CxThreadPriority`、`OpenUrlInPlace`
- **`display_context`**：`DisplayContext`、`SystemBarAppearance`
- **`draw_list`**：`CxDrawCall`、`CxDrawItem`、`CxDrawListPool`、`CxRectArea`、`DrawList`、`DrawListId`
- **`draw_pass`**：`DrawPass`、`DrawPassId`、`DrawPassClear*`、`CxDrawPass*`、`ScriptDrawPass`
- **`event`**：所有事件类型（35+ 种）：鼠标、键盘、触控、拖放、窗口、文本输入、XR、定时器等
- **`gl_render_bridge`**：`GlApi`、`GlRenderBridge`
- **`ime`**：`AutoCapitalize`、`AutoCorrect`、`InputMode`、`ReturnKeyType`、`SoftKeyboardConfig`、`TextInputConfig`
- **`log`**：日志宏
- **`share_bytes`**：`MappedBytes`、`SharedBytes`、`SharedBytesStats`
- **`texture`**：纹理类型
- **`ui_runner`**：`UiRunner`、`DeferCallback`
- **`uniform_buffer`**：`UniformBuffer`、`UniformBufferId`
- **`web_socket`**：`WebSocket`、`WebSocketMessage`
- **`window`**：`WindowHandle`、`WindowId`、`WindowVisuals`、`WindowBackdrop`、`WindowIcon`、`MacosWindowConfig`、`ScriptWindowHandle`
- **`xr_tsdf`**：XR TSDF 类型
- **`makepad_math`**：数学类型（`Vec2`、`Vec3`、`Mat4` 等）和 `makepad_micro_serde`
- 以及 `audio`、`game_input`、`midi`、`os`、`thread`、`video` 的全部内容
- **`media_*`** 和 **`playback_session`** / **`video_session`**：媒体播放管道类型
- **`permission`** 和 **`gpu_info`**：权限和 GPU 性能查询

### 条件编译导出

```rust
#[cfg(target_arch = "wasm32")]
pub use makepad_wasm_bridge;

#[cfg(any(target_os = "macos", target_os = "ios", target_os = "tvos"))]
pub use makepad_objc_sys;

#[cfg(target_os = "windows")]
pub use ::windows;
```

- **Wasm32**：导出 `makepad_wasm_bridge`，让应用代码可以直接访问 WebAssembly 桥接。
- **Apple 平台**：导出 `makepad_objc_sys`，供需要直接调用 Objective-C 运行时 API 的代码使用。
- **Windows**：导出 `windows` crate，使 Windows 平台上的原生 Windows API 调用可用。

---

## 导出策略设计考量

1. **用户便利性**：应用开发者只需 `use makepad_platform::*` 即可获得几乎所有常用类型，无需逐层深入模块路径。
2. **内部封装**：关键实现细节（如 `cx` 内部宏、`draw_shader`、`id_pool` 等）保持私有，不暴露给外部使用者。
3. **宏支持**：`log` 和 `cx` 用 `#[macro_use]` 声明，使宏在 crate 内所有模块中可用。
4. **条件平台适配**：通过条件编译导出平台特有的依赖，确保跨平台兼容性。
5. **类型安全**：`app_main` 模块的 `resolve_studio_http` 和 `should_run_stdin_loop_from_env` 被显式导出，供 Studio 外部工具集成使用。
