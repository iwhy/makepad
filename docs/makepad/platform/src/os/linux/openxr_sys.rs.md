# openxr_sys.rs — OpenXR FFI Bindings & Xr Type Definitions

**File:** `platform/src/os/linux/openxr_sys.rs` (5305 行)  
**核心用途:** OpenXR 运行时的完整 FFI 绑定层：Xr* 句柄/结构体定义、函数指针类型、`LibOpenXrLoader`（加载器函数表）和 `LibOpenXr`（实例函数表）的动态符号加载，以及辅助宏（`get_proc_addr!`、`bitmask!`）。

---

## FFI Bindings

| 类型 | 说明 |
|------|------|
| `LibOpenXrLoader` | 加载器级别函数表：`xrCreateInstance`、`xrGetInstanceProcAddr`、`xrEnumerateInstanceExtensionProperties`、`xrEnumerateApiLayerProperties`、`xrInitializeLoaderKHR`；持有 `ModuleLoader` 保持 `.so` 存活 |
| `LibOpenXr` | 实例级别函数表：约 60 个 OpenXR 函数指针，涵盖会话、交换链、空间、动作、手柄追踪、环境深度、透视、注视点渲染、空间锚点、同位置发现等扩展 |

### 函数指针类型示例

`TxrCreateInstance`, `TxrGetSystem`, `TxrCreateSession`, `TxrWaitFrame`, `TxrBeginFrame`, `TxrEndFrame`, `TxrCreateSwapchain`, `TxrLocateHandJointsEXT`, `TxrCreateSpatialAnchorFB`, `TxrCreateEnvironmentDepthProviderMETA` 等。

### 辅助宏

| 宏 | 说明 |
|-----|------|
| `get_proc_addr!` | 通过 `xrGetInstanceProcAddr` 加载必需函数指针，失败返回 `Err` |
| `get_optional_proc_addr!` | 同上但返回 `Option`，用于 Foveation、Display Refresh Rate 等可选扩展 |
| `bitmask!` | 为位掩码结构体添加 `contains()` 和 `BitOr` 实现（如 `XrInstanceCreateFlags`、`XrSpaceLocationFlags`） |
| `xr_result_check!` | 检查 `XrResult` 并 `unwrap_or` 默认值 |
| `impl_xr_enum!` | 为枚举添加 `try_from` 和 `from` 转换（如 `XrStructureType` <=> `LiveId`） |

---

## Xr* 结构体定义（关键）

| 结构体 | 说明 |
|--------|------|
| `XrInstance(u64)` | OpenXR 实例句柄 |
| `XrSession(u64)` | XR 会话句柄 |
| `XrSpace(u64)` | 参考空间句柄 |
| `XrActionSet(u64)`, `XrAction(u64)` | 输入动作集/动作句柄 |
| `XrSwapchain(u64)` | 交换链条柄 |
| `XrHandTrackerEXT(u64)` | 手柄追踪器句柄 |
| `XrEnvironmentDepthProviderMETA(u64)` | 环境深度提供者句柄 |
| `XrSpatialAnchorFB(u64)` | 空间锚点句柄 |
| `XrInstanceCreateInfo` | 实例创建信息（含 `application_info`、扩展/层列表） |
| `XrSessionCreateInfo` | 会话创建信息（含图形绑定） |
| `XrSwapchainCreateInfo` | 交换链创建参数（宽/高/格式/面数/用途标志等） |
| `XrView` | 视图（位置、姿态、FOV） |
| `XrViewState` | 视图状态标志 |
| `XrFrameState` | 帧状态（`predicted_display_time`、`should_render` 等） |
| `XrSpaceLocation` | 空间定位结果（姿态/位置标志 + `XrPosef`） |
| `XrEventDataBuffer` | 事件轮询缓冲区（`ty` + `varying` 4000 字节） |
| `XrEventDataSessionStateChanged` | 会话状态变更事件 |
| `XrEventDataDisplayRefreshRateChangedFB` | 显示刷新率变更事件 |
| `XrEventDataReferenceSpaceChangePending` | 参考空间重设事件 |
| `XrHandJointLocationEXT` | 手部关节位置（26 关节 × 位置/姿态/半径） |
| `XrEnvironmentDepthProviderCreateInfoMETA` | 环境深度提供者创建信息 |
| `XrEnvironmentDepthSwapchainCreateInfoMETA` | 环境深度交换链创建信息 |
| `XrPassthroughCreateInfoFB` | 透视创建信息 |
| `XrSpatialAnchorCreateInfoFB` | 空间锚点创建信息 |
| `XrSpaceQueryInfoFB` | 空间查询信息 |
| `XrSpaceSaveInfoFB` | 空间持久化保存信息 |

---

## Implementation Details

- **动态加载:** `LibOpenXrLoader::try_load()` 使用 `ModuleLoader::load("libopenxr_loader.so")` 运行时加载，不要求编译时链接
- **两阶段初始化:** `LibOpenXrLoader` 从加载器获取基本函数，创建实例后再通过 `xrGetInstanceProcAddr` 加载所有实例函数
- **可选 vs 必需:** 基础 XR 函数使用 `get_proc_addr!`（不可为空），扩展函数（Foveation、DisplayRefreshRate）使用 `get_optional_proc_addr!`
- **设备枚举:** `xr_array_fetch!` 辅助宏用于两阶段枚举（先查容量再填充）
- **Vulkan 借用:** 定义 `VkInstance`/`VkDevice` 等 Vulkan 透传类型，供 `XR_KHR_vulkan_enable2` 扩展使用
- **结构体布局:** 所有 Xr 结构体遵循标准 C 布局（`#[repr(C)]`），含 `ty` + `next` + `XrResult`

---

## Platform Integration

- **Android:** 通过 `XrLoaderInitInfoAndroidKHR` 传递 Java VM 和 Activity Context
- **OpenXR 扩展:** 函数指针覆盖 Core 规范 + FB/META/EXT/KHR 扩展，包括空间锚点（`FB`）、同位置发现（`META`）、环境深度（`META`）、手柄追踪（`EXT`）、注视点渲染（`FB`）、透视（`FB`）
- **与 `openxr.rs` 联动:** `LibOpenXr` 被 `CxOpenXr::create_instance` 和 `CxOpenXrFrame::begin_frame` 使用
- **与 `openxr_input.rs` 联动:** 动作/动作集/手柄追踪的创建与状态查询
- **与 `openxr_anchor.rs` 联动:** 空间锚点的创建/查询/持久化/共享
