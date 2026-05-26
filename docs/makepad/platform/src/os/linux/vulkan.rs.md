# vulkan.rs — Android Vulkan Rendering Backend

**File:** `platform/src/os/linux/vulkan.rs` (7118 行)  
**核心用途:** Android 平台 Vulkan 渲染后端，封装 `ash` crate：Vulkan 实例/设备/交换链创建、着色器管线编译、几何/纹理资源管理、OpenXR Multi-View 与单视图渲染、环境深度网格 GPU 计算与 CPU 回读、GPU 时间戳查询。仅在 `cfg(target_os = "android")` 下编译。

---

## Types / Structs

| 类型 | 说明 |
|------|------|
| `CxVulkan` | 顶层 Vulkan 状态：Instance、Device、Queue、Surface、Swapchain、RenderPass、Pipelines、Geometries、Textures、In-Flight Frames、OpenXR 会话数据 |
| `VulkanBuffer` | Buffer + DeviceMemory + 大小 |
| `VulkanGeometryResource` | 顶点 Buffer + 索引 Buffer |
| `VulkanTextureResource` | Image + Memory + View + Face Views（CubeMap）、硬件 Buffer（AHardwareBuffer）、YCbCr 转换、Sampler |
| `VulkanTextureUpload` | 上传数据缓冲（含偏移/尺寸/层数） |
| `VulkanDrawPacket` | 单次绘制调用数据：Shader 索引、几何体 ID、深度/混合/面剔除开关、Instance 数据、Uniform 绑定、纹理 ID |
| `VulkanPipeline` | Pipeline（写/不写深度）+ Layout + DescriptorSetLayout + Sampler 列表 |
| `VulkanRenderPassKey` | RenderPass 哈希键（颜色格式列表 + 深度格式） |
| `VulkanPipelineKey` | Pipeline 缓存键（Shader 索引 + 变体 + RenderPass + AlphaBlend + BackfaceCulling） |
| `FrameResources` | 每帧资源池：Buffers、DescriptorPools、PacketBuffer |
| `VulkanXrInFlightFrame` | OpenXR 帧状 Flight 资源：CommandBuffer、Fence、时间戳 QueryPool |
| `CxVulkanOpenXrMultiviewTarget` | Multi-View 帧缓冲（Framebuffer + ColorView + DepthTarget + FragmentDensityView） |
| `CxVulkanOpenXrSwapchainImage` | OpenXR 交换链图像 + Multi-View Target |
| `CxVulkanOpenXrDepthImage` | OpenXR 深度图像 + 双视图 + Multi-View View |
| `CxVulkanOpenXrSessionData` | OpenXR 会话图形数据（宽/高/格式/颜色图像列表/深度图像列表/回读缓冲） |
| `OpenXrVulkanRepaintStats` | 重绘性能统计（等待/纹理预备/录制/提交 + 各项计数器） |
| `VulkanDrawStats` | 绘制统计（命中/跳过各原因计数） |
| `ImportedYuvPlaneLayout` | 导入 YUV 平面布局配置 |

---

## Key Methods (impl CxVulkan)

### 初始化和生命周期

| 方法 | 说明 |
|------|------|
| `new()` | 标准 Android Vulkan 初始化：加载 `ash::Entry` → 枚举 Layer/Extension → 创建 Instance → 创建 Android Surface → 选择物理设备 → 创建 Device → 创建 Semaphore/Fence/CommandPool → `recreate_swapchain()` |
| `new_from_openxr()` | OpenXR 引导的 Vulkan 初始化：通过 `xrCreateVulkanInstanceKHR` 与 `xrCreateVulkanDeviceKHR` 由 XR 运行时创建 Instance/Device，然后自建渲染结构 |
| `recreate_swapchain()` | 重建交换链（查询 Surface 能力、选择格式/呈现模式、创建 Image/ImageView/Framebuffer/DepthTarget） |
| `destroy()` | 完整的资源清理：等待设备空闲、销毁所有 In-Flight Frames/Pipelines/Textures/Geometries/FrameResources/RenderPass/DepthTargets/ImageView/Swapchain/Device/Instance |
| `try_enable_debug_messenger()` | 可选启用 Vulkan 验证层 Debug Messenger |
| `pick_device_and_queue_family()` | 选择物理设备（优先集成 GPU）和图形队列族 |
| `create_surface()` | 通过 `ASurfaceWindow_createDisplay` 创建 Android Surface |

### 标准绘制

| 方法 | 说明 |
|------|------|
| `draw_frame()` | 标准 2D 窗口绘制：等待 In-Flight Fence → Acquire 交换链图像 → 解析绘制列表 → 执行 CommandBuffer → Present |
| `prepare_command_buffer()` / `record_draw_commands()` | 录制绘制命令（清除、动态状态设置、Pipeline 绑定、Descriptor Set 绑定、DrawIndexed/Draw 调用） |
| `compile_shaders()` | 编译 Makepad Shader 到 SPIR-V（通过 `vulkan_naga.rs`）→ 创建 Pipeline 并缓存 |
| `update_texture()` / `update_texture_2d_array()` | 上传 2D/Array/Cube/YCbCr 纹理到 GPU |
| `update_geometry()` | 上传顶点/索引数据到 GPU |
| `uniforms` / `uniform_textures` | Uniform Buffer 与纹理 Descriptor Set 绑定 |

### OpenXR 渲染

| 方法 | 说明 |
|------|------|
| `create_xr_swapchain_images()` | 创建 OpenXR 交换链的 Vulkan Image 包装（Color/Depth/FragmentDensity） |
| `destroy_xr_swapchain_images()` | 销毁 OpenXR 交换链的 Vulkan 资源 |
| `draw_frame_to_xr()` | OpenXR 帧绘制主入口：等待 In-Flight → 解析绘制列表 → 执行 CommandBuffer（渲染到 OpenXR 图像） |
| `create_xr_session_data()` | 创建 OpenXR 会话的 Vulkan 配置（Multi-View RenderPass、Framebuffer 等） |
| `destroy_xr_session_data()` | 销毁 OpenXR 会话的 Vulkan 资源 |
| `submit_depth_mesh_job()` | 环境深度网格 GPU 计算 + CPU 回读管线 |
| `resize_xr_projection()` | OpenXR 投影缩放重设 |

### 资源管理

| 方法 | 说明 |
|------|------|
| `prune_stale_geometry_resources()` | 清理失效 Geometry 资源 |
| `create_geometry_resource()` / `destroy_geometry_resource()` | 几何体资源生命周期 |
| `create_texture_resource()` / `destroy_texture_resource()` | 纹理资源生命周期 |
| `create_render_pass()` | 创建/缓存 RenderPass（按颜色+深度格式） |
| `create_framebuffer()` | 创建帧缓冲 |
| `get_or_create_pipeline()` | 获取/创建 Pipeline（按 Shader+变体+Pass+混合+面剔除） |

---

## Implementation Details

- **双模式渲染:** 标准 Android 窗口（`draw_frame`）和 OpenXR（`draw_frame_to_xr`）共用底层资源，但使用不同交换链和 CommandBuffer
- **Multi-View:** OpenXR 使用 `VK_KHR_multiview` 扩展一次性渲染双眼
- **Fragment Density Map:** 支持 VK_FB_foveation_vulkan 提供的注视点渲染密度贴图
- **GPU 时间戳:** 通过 `VK_QUERY_PIPELINE_STATISTIC` 和 Timestamp Query 收集 GPU 帧时间
- **AHardwareBuffer 导入:** 支持 YCbCr 硬件缓冲的直接导入（`VK_ANDROID_external_memory_android_hardware_buffer`）
- **环境深度网格:** 通过 OpenXR `XR_META_environment_depth` 扩展获取环境深度图像 → GPU Compute Shader 生成 Mesh → CPU 回读
- **Pipeline 缓存:** 以 `VulkanPipelineKey` 为键哈希缓存，避免重复创建
- **离线资源清理:** `prune_stale_geometry_resources` 每帧检查 Geometry 代际号，释放已删除的 GPU 资源

---

## Platform Integration

- **Android 专用:** `#[cfg(target_os = "android")]` 条件编译，依赖 `ash`、`ndk-sys`、`nativewindow` 库
- **与 OpenXR 联动:** 通过 `openxr_sys.rs` 的 `XrVulkanInstanceCreateInfoKHR`/`XrVulkanDeviceCreateInfoKHR` 由 XR 运行时创建 Vulkan 资源
- **与 `vulkan_naga.rs` 联动:** 调用 Naga 将 SPIR-V 编译为 Vulkan Shader Module
- **纹理上传:** 支持类别 `TextureCategory::XrEnvironmentDepth` 用于 OpenXR 深度图像传递
