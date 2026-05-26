# `cx.rs` — Cx 主上下文结构体

## 概述

`cx.rs` 是 Makepad 框架中最核心的数据结构文件，定义了全局上下文 `Cx`（约 60+ 字段）。`Cx` 是贯穿整个框架的"上帝对象"，管理窗口、渲染通道、纹理、着色器、几何数据、事件、定时器、脚本虚拟机等全部子系统。所有 widget、绘图操作、平台交互都通过 `&mut Cx` 完成。

---

## 类型与结构体

### `PendingCameraPlayback`（第 49-58 行）

等待摄像头权限确认的相机播放请求。在权限未授予时，播放参数暂存于此，权限响应后转为实际 `CxOsOp::PrepareVideoPlayback` 操作。包含 permission、video_id、source、camera_preview_mode、external_texture_id、texture_id、autoplay、should_loop 等字段。

### `Cx`——核心上下文结构体（第 60-188 行）

#### 脚本与调试
| 字段 | 类型 | 说明 |
|------|------|------|
| `script_vm` | `Option<Box<ScriptVmBase>>` | Makepad 脚本虚拟机，驱动 `script_mod!` 动态 UI 系统的运行时 |
| `script_data` | `CxScriptData` | 脚本相关的附加数据（资源、crate manifests、热重载状态） |
| `package_root` | `Option<String>` | 包根路径，用于资源文件查找 |
| `debug_trace_active` | `bool` | 是否激活调试追踪 |
| `debug` | `Debug` | 调试信息收集器 |

#### 平台信息
| 字段 | 类型 | 说明 |
|------|------|------|
| `os_type` | `OsType` | 当前操作系统类型（Windows/MacOS/iOS/Android等） |
| `in_makepad_studio` | `bool` | 是否运行在 Makepad Studio IDE 内部 |
| `gpu_info` | `GpuInfo` | GPU 信息（厂商、型号、特性） |
| `xr_capabilities` | `XrCapabilities` | XR（AR/VR）能力描述 |
| `cpu_cores` | `usize` | CPU 核心数 |

#### 图形资源池
| 字段 | 类型 | 说明 |
|------|------|------|
| `null_texture` | `Texture` | 空纹理（4x4 全透明），作为默认/占位纹理 |
| `null_cube_texture` | `Texture` | 空立方体贴图 |
| `windows` | `CxWindowPool` | 窗口池，管理所有打开的窗口 |
| `passes` | `CxDrawPassPool` | 渲染通道池，管理渲染流程 |
| `draw_lists` | `CxDrawListPool` | 绘制列表池，管理绘制命令 |
| `draw_matrices` | `CxDrawMatrixPool` | 绘制矩阵池 |
| `textures` | `CxTexturePool` | 纹理池，管理 GPU 纹理 |
| `uniform_buffers` | `CxUniformBufferPool` | 统一缓冲区池 |
| `geometries` | `CxGeometryPool` | 几何数据池 |
| `draw_shaders` | `CxDrawShaders` | 绘制着色器集合 |

#### 事件与渲染状态
| 字段 | 类型 | 说明 |
|------|------|------|
| `new_draw_event` | `DrawEvent` | 累积的重绘请求，在事件循环中被处理 |
| `redraw_id` | `u64` | 全局递增重绘 ID，用于判断 Area 有效性 |
| `repaint_id` | `u64` | 全局递增刷帧 ID |
| `event_id` | `u64` | 全局递增事件 ID |
| `timer_id` | `u64` | 全局递增定时器 ID |
| `next_frame_id` | `u64` | 全局递增下一帧 ID |
| `permissions_request_id` | `i32` | 权限请求 ID |

#### 输入系统
| 字段 | 类型 | 说明 |
|------|------|------|
| `keyboard` | `CxKeyboard` | 键盘状态（焦点、IME、修饰键） |
| `fingers` | `CxFingers` | 触控/手指状态（触摸、滑动锁定、手势识别） |
| `ime_area` | `Area` | 当前 IME（输入法）关联的交互区域 |
| `keyboard_shift` | `f64` | 键盘弹出时窗口位移量（移动端） |
| `drag_drop` | `CxDragDrop` | 拖放状态 |

#### 平台操作/事件队列
| 字段 | 类型 | 说明 |
|------|------|------|
| `platform_ops` | `Vec<CxOsOp>` | 平台操作命令队列，延迟到 OS 层处理 |
| `pending_camera_playbacks` | `Vec<PendingCameraPlayback>` | 等待权限的摄像头播放请求 |
| `new_next_frames` | `HashSet<NextFrame>` | 新注册的下一帧请求 |
| `new_actions` | `ActionsBuf` | 新累积的 widget 动作 |
| `dependencies` | `HashMap<String, CxDependency>` | 文件依赖表，运行时资源加载 |
| `triggers` | `HashMap<Area, Vec<Trigger>>` | 按 Area 索引的触发器 |
| `action_receiver` | `Receiver<ActionSend>` | 接收跨线程动作的通道 |

#### 系统服务
| 字段 | 类型 | 说明 |
|------|------|------|
| `os` | `CxOs` | 操作系统抽象层 |
| `event_handler` | `Option<Box<dyn FnMut(&mut Cx, &Event)>>` | 顶层事件处理器回调 |
| `globals` | `Vec<(TypeId, Box<dyn Any>)>` | 类型擦除的全局单例存储 |
| `components` | `ComponentRegistries` | 组件工厂注册表 |
| `self_ref` | `Option<Rc<RefCell<Cx>>>` | Cx 的自身引用（用于 `CxRef`） |
| `in_draw_event` | `bool` | 是否正在处理绘制事件 |
| `display_context` | `DisplayContext` | 显示上下文（安全区域、系统栏外观） |
| `executor` | `Option<Executor>` | Futures 执行器 |
| `spawner` | `Spawner` | Futures 生成器 |
| `net` | `Arc<NetworkRuntime>` | 网络运行时（HTTP、WebSocket） |
| `performance_stats` | `PerformanceStats` | 性能统计 |
| `studio_http` | `String` | Studio HTTP 地址 |

#### 热重载与脚本更新
| 字段 | 类型 | 说明 |
|------|------|------|
| `pending_script_reapply` | `bool` | 请求下一次事件循环触发 `Event::ScriptReapply`（保留运行时 `script_eval!` 覆盖） |
| `pending_live_edit_request` | `bool` | 请求下一次事件循环触发 `Event::LiveEdit`（重新运行 `script_mod` + 完全 reload） |
| `pending_window_geom_changes` | `Vec<WindowGeomChangeEvent>` | 等待调度的事件队列（窗口几何变更） |

#### Widget 树调试/快照
| 字段 | 类型 | 说明 |
|------|------|------|
| `screenshot_requests` | `Vec<ScreenshotRequest>` | Studio 屏幕截图请求 |
| `widget_tree_dump_requests` | `Vec<u64>` | Widget 树转储请求 |
| `widget_snapshot_requests` | `Vec<u64>` | Widget 快照请求 |
| `widget_query_invalidation_event` | `Option<u64>` | 触发 widget 查询缓存失效的事件 ID |
| `widget_tree_ptr` | `*mut ()` | 不透明 widget 树根指针 |
| `widget_tree_dump_callback` | `Option<fn(&Cx) -> String>` | 生成 widget 树转储的回调 |
| `widget_query_callback` | `Option<fn(&Cx, &str) -> Vec<String>>` | 执行 widget 查询的回调 |
| `widget_snapshot_callback` | `Option<fn(&Cx) -> Vec<WidgetSnapshot>>` | 生成 widget 快照的回调 |

---

### `CxRef`——Cx 引用包装（第 190-191 行）

`pub struct CxRef(pub Rc<RefCell<Cx>>)` — 通过 `Rc<RefCell<Cx>>` 实现的可克隆引用，用于在不需要 `&mut Cx` 的场景中持有 Cx 的共享访问。

### `CxDependency`——依赖数据（第 193-195 行）

保存文件依赖的二进制数据，类型为 `Option<Result<Rc<Vec<u8>>, String>>`，加载成功时为 `Ok(Rc<Vec<u8>>)`，失败时为 `Err(String)`。

---

### `OsType`——操作系统类型枚举（第 266-283 行）

`#[derive(Clone, Debug, Script, ScriptHook)]` — 通过 Script 系统暴露给脚本层。

| 变体 | 说明 |
|------|------|
| `Unknown` | 未知 |
| `Windows` | Windows |
| `Macos` | macOS |
| `Ios(IosParams)` | iOS/tvOS，携带 `IosParams` 参数 |
| `Android(AndroidParams)` | Android，携带 `AndroidParams` |
| `OpenHarmony(OpenHarmonyParams)` | OpenHarmony |
| `LinuxWindow(LinuxWindowParams)` | Linux 桌面窗口 |
| `LinuxDirect` | Linux 直接模式（无窗口管理器） |
| `Web(WebParams)` | Web 平台 |

**平台参数结构体：**
- `AndroidParams`（第 196-214 行）：cache_path、data_path、density、is_emulator、has_xr_mode、android_version、build_number、kernel_version
- `IosParams`（第 216-224 行）：data_path、device_model、system_version
- `OpenHarmonyParams`（第 226-240 行）：files_dir、cache_dir、temp_dir、device_type、os_full_name、display_density
- `WebParams`（第 242-258 行）：protocol、host、hostname、pathname、search、hash、small_font_aliases
- `LinuxWindowParams`（第 260-264 行）：custom_window_chrome

**方法：**
| 方法 | 说明 |
|------|------|
| `is_single_window()` | 判断是否为单窗口平台（Web/iOS/Android/LinuxDirect）。返回值决定是否允许创建多个窗口 |
| `is_web()` | 是否为 Web 平台 |
| `has_xr_mode()` | 当前平台是否支持 XR 模式（仅 Android） |
| `get_cache_dir()` | 获取平台缓存目录路径（Android/OpenHarmony） |
| `get_data_dir()` | 获取平台数据目录路径（Android/iOS/OpenHarmony） |

---

### `XrCapabilities`——XR 能力（第 285-289 行）

```rust
pub struct XrCapabilities {
    pub ar_supported: bool,
    pub vr_supported: bool,
}
```

---

## `Cx::new()`——构造函数（第 339-487 行）

**实现逻辑：**

1. **安装终止信号处理器**：在 Linux/macOS/Windows 上安装 SIGTERM/SIGINT 信号处理器，用于优雅退出
2. **创建空纹理**：在 `CxTexturePool` 中分配一个 4x4 全透明 BGRA 纹理作为 `null_texture`，以及一个 6 面全透明的立方体贴图作为 `null_cube_texture`。这些纹理作为默认绑定，避免 GPU 访问未初始化纹理
3. **初始化 futures 执行器**：创建 `Executor` + `Spawner`，为 `makepad_futures` 提供异步运行时
4. **初始化全局动作通道**：创建 `std::sync::mpsc` 通道，将 sender 存入全局静态 `ACTION_SENDER_GLOBAL`。跨线程代码可通过此通道向主线程发送 `ActionSend`
5. **安装网络后端 shim**：在 wasm32 上安装 Web 网络后端 shim；在 Android 上安装 Android 网络后端 shim，在 OS 层创建前截获网络调用
6. **创建网络运行时**：`NetworkRuntime` 使用默认 DNS/HTTP 配置启动。它的 wake function 设为向 UI 线程发信号，确保异步网络事件能被主循环处理
7. **创建并初始化脚本虚拟机**：创建 `ScriptVm`，注入 `ScriptStd`（包含网络运行时）。调用 `crate::script::script_mod()` 注册所有平台级脚本模块、widget 定义、theme、prelude。初始化完成后将 `ScriptVmBase` 从栈上移出，存入 `script_vm` 字段
8. **返回完整 Cx**：将所有子系统（窗口池、渲染通道池、纹理池、几何池、着色器集合、定时器、事件状态、输入系统、性能统计、widget 树回调等）初始化并组装为最终的 `Cx` 结构体
