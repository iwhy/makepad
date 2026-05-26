# openxr.rs — OpenXR XR Runtime Session & Frame Loop

**File:** `platform/src/os/linux/openxr.rs` (1408 行)  
**核心用途:** 实现 OpenXR XR 运行时的完整会话管理、事件处理、帧循环与绘制调度，支持 OpenGL ES Multi-View 和 Vulkan 后端，包括环境深度网格回读、显示刷新率变更、参考空间重设等运行时逻辑。

---

## Types / Structs

| 类型 | 说明 |
|------|------|
| `CxOpenXr` | 顶层 OpenXR 状态聚合：Loader 实例、`LibOpenXr` 函数表、`XrInstance`、`XrSystemId`、`CxOpenXrSession` |
| `CxOpenXrFrame` | 单帧封装（`openxr_opengl.rs` / `openxr_vulkan.rs`），包含 `frame_state`、`projection_views`、`depth_image`、`color_swapchain`、`end_frame()` 等 |
| `CxOpenXrOptions` | 会话创建选项（缓冲比例、深度启用等） |

---

## Key Methods (impl Cx)

| 方法 | 说明 |
|------|------|
| `openxr_render_loop()` | 主渲染循环入口：清空 Java 消息队列 → 处理 XR 事件 → 处理其他事件 → 处理绘制 |
| `openxr_handle_events()` | 轮询 `xrPollEvent`，处理会话状态变化（IDLE→READY→STOPPING→EXITING）、显示刷新率变更、参考空间重设 |
| `openxr_handle_drawing()` | 帧循环核心：调用 `CxOpenXrFrame::begin_frame()` → 更新输入 → 派发 `XrUpdate` 事件 → 派发 draw 事件 → 编译 Shader → `openxr_handle_repaint()` → 深度网格提交 → `end_frame()` → 投影层缩放重设 |
| `openxr_handle_repaint()` | 遍历绘制 Pass 列表：Vulkan 后端走 `openxr_draw_pass_to_vulkan`，GLES 后端走 `openxr_draw_pass_to_multiview` |

## Key Methods (impl CxOpenXr)

| 方法 | 说明 |
|------|------|
| `create_instance()` | 加载 `libopenxr_loader.so` → 初始化 Loader → 枚举扩展 → 创建 `XrInstance` → 加载 `LibOpenXr` 函数表 → 查询系统属性 → 创建 Vulkan/GLES 绑定 |
| `create_session()` | 为当前 system 创建 `XrSession`（含图形绑定环境），更新显示刷新率，获取本地 Anchor |
| `destroy_session()` | 销毁会话，清理后端子 |
| `create_vulkan_backend()` | 委托 `CxVulkan::new_from_openxr()` 创建 Vulkan 后端 |
| `resize_projection_layer()` | 缩放 XR 投影层（渲染缓冲比例变更） |
| `destroy_instance()` | 依次销毁会话、所有 Swapchain/Space/ActionSet/HandTracker/Session/Instance |
| `compute_android_xr_options()` | 计算当前交换链配置选项（宽度、高度、缓冲比例、色彩深度格式） |

---

## Implementation Details

- **会话状态机:** 通过 `XrEventDataSessionStateChanged` 驱动：`READY→begin_session()`、`STOPPING→end_session()`、`EXITING→清理`
- **Vulkan/GLES 双后端:** 通过 `#[cfg(use_vulkan)]` / `#[cfg(not(use_vulkan))]` 分别编译，扩展列表不同
- **深度网格回读:** `OPENXR_DEPTH_MESH_READBACK_ENABLED` 控制环境深度 Mesh 的 CPU 回读，通过 `submit_depth_mesh_job()` 提交
- **显示刷新率:** 跟踪 `XrEventDataDisplayRefreshRateChangedFB`，更新 `active_display_refresh_rate_hz` 与 `effective_frame_time_ms`
- **参考空间重设:** `XrEventDataReferenceSpaceChangePending` 触发 TSDF 深度 volume 重设
- **CPU 性能分解:** 每帧收集 `XrFrameCpuBreakdown`：update_prepare、update_dispatch、draw_event、compile、repaint、depth_readback、end_frame、resize 各阶段毫秒数

---

## Platform Integration

- **Android:** 通过 Android JNI 消息驱动渲染循环，使用 `ANativeWindow` 创建 EGL/Vulkan Surface
- **OpenXR 扩展:** 请求式启用（如 `XR_FB_passthrough`、`XR_META_environment_depth`、`XR_FB_foveation` 等），缺失时仅报 warning
- **与 `openxr_input.rs` 联动:** 每帧调用 `session.new_xr_update_event()` 同步动作状态
- **与 `openxr_anchor.rs` 联动:** 转发未识别事件到 `session.handle_anchor_events()`
