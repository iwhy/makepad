# openxr_vulkan.rs

One-liner (EN): OpenXR session creation and rendering for the Vulkan backend — swapchain management, fixed foveation, color format negotiation, and depth mesh pipeline integration.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/openxr_vulkan.rs` (589 行)
- **核心作用**: 实现 OpenXR 的 Vulkan 后端：创建和配置 Vulkan XR 会话、颜色/深度交换链管理、固定注视点渲染（Fixed Foveation）、缓冲区缩放调整、Vulkan 渲染目标创建，以及深度网格管线集成。

## 类型/结构体

### `CxOpenXrVulkanSession` — Vulkan XR 会话数据

| 字段 | 类型 | 说明 |
|------|------|------|
| `_color_images` | `Vec<XrSwapchainImageVulkanKHR>` | 颜色交换链图像 |
| `_depth_images` | `Vec<XrSwapchainImageVulkanKHR>` | 深度交换链图像 |
| `render_targets` | `CxVulkanOpenXrSessionData` | Vulkan 渲染目标 |
| `depth_mesh_pipeline` | `CxOpenXrDepthMeshPipeline` | 深度网格管线 |
| `retired_projection_layers` | `Vec<RetiredOpenXrVulkanProjectionLayer>` | 已退役的投影层（用于 resize 清理） |

### `RetiredOpenXrVulkanProjectionLayer` — 已退役投影层

| 字段 | 说明 |
|------|------|
| `color_swap_chain` | 旧的 XrSwapchain |
| `render_targets` | 旧的 Vulkan 渲染目标 |

## 关键方法

### Cx 方法

| 方法 | 说明 |
|------|------|
| `openxr_draw_pass_to_vulkan(draw_pass_id, frame) -> Result<OpenXrVulkanRepaintStats, String>` | 渲染 OpenXR Vulkan pass：设置相机 uniform → 借用 Vulkan 后端 → `vulkan.draw_openxr_view()` → [Android] 可选的 Studio 帧读取 |

### CxOpenXrVulkanSession 方法

| 方法 | 说明 |
|------|------|
| `submit_depth_mesh_job(vulkan, frame, depth_image_index) -> Result<(), String>` | 提交深度网格作业到 `depth_mesh_pipeline` |

### CxOpenXrSession 方法 (Vulkan)

| 方法 | 说明 |
|------|------|
| `create_session_vulkan(xr, system_id, instance, vulkan, options) -> Result<Self, String>` | **创建 Vulkan XR 会话**：验证图形设备一致性 → 创建 session → 描述立体配置 → 创建透传和深度 → 创建投影层资源 (`create_vulkan_projection_layer_resources`) → 启动深度提供器 → 创建 inputs |
| `destroy_session_vulkan(xr, vulkan)` | 销毁 Vulkan 资源：清理退役投影层 → 销毁当前渲染目标 → 清空 TSDF store |
| `resize_projection_layer_vulkan(xr, vulkan, options) -> Result<(), String>` | 调整投影层尺寸：按 `recommended * buffer_scale` 计算新尺寸 → 创建新交换链和渲染目标 → 旧资源放入 `retired_projection_layers` 延迟销毁 |
| `create_vulkan_projection_layer_resources(xr, session, vulkan, options, width, height, depth_images, depth_width, depth_height) -> Result<(XrSwapchain, Vec<...>, CxVulkanOpenXrSessionData), String>` | **创建投影层资源**：枚举交换链格式 → 选择颜色格式 → 创建交换链（可选固定注视点） → 枚举交换链图像 → 调用 `vulkan.create_openxr_session_data()` 创建 Vulkan 渲染目标 |
| `pick_vulkan_color_format(supported_formats, preferred_format) -> Option<vk::Format>` | 选择 Vulkan 颜色格式：首选格式 → BGRA8/RGBA8 UNORM/SRGB 降级 |
| `desired_fixed_foveation_level(level) -> Option<XrFoveationLevelFB>` | 映射配置级别：0=None, 1=LOW, 2=MEDIUM, 其余 HIGH |
| `try_enable_fixed_foveation(xr, session, swapchain, level) -> Result<(), String>` | 启用固定注视点：创建 Foveation Profile → `xrUpdateSwapchainFB` 设置 → 销毁 Profile |

## 实现细节

### Vulkan 设备一致性检查

```rust
xrGetVulkanGraphicsDevice2KHR → 返回 runtime 期望的物理设备
比较 runtime_physical_device != vulkan.physical_device_handle()
若不匹配 → 返回 Err "device mismatch"
```

确保 OpenXR runtime 和 Makepad 使用相同的 Vulkan 物理设备。

### 颜色格式选择优先级

1. `vulkan.swapchain_format()` — 根据 surface 配置的首选格式
2. 降级链（逐个检查 runtime 支持情况）:
   - `B8G8R8A8_UNORM`
   - `R8G8B8A8_UNORM`
   - `B8G8R8A8_SRGB`
   - `R8G8B8A8_SRGB`

### 固定注视点 (Fixed Foveation)

```
配置: options.fixed_foveation_level (0=禁用, 1/2/3=低/中/高)

启用条件:
  1. level > 0
  2. vulkan.supports_openxr_fixed_foveation()
  3. xrCreateFoveationProfileFB / xrUpdateSwapchainFB 函数指针非空

流程:
  1. 在交换链创建信息中添加 XrSwapchainCreateInfoFoveationFB
  2. 创建 FoveationProfile → 配置级别/偏移/动态模式
  3. xrUpdateSwapchainFB → 应用配置
  4. 销毁 FoveationProfile

失败处理:
  - 不使会话创建失败，仅记录 warning
  - 回退到无注视点渲染

注视点交换链图像枚举:
  使用 XrSwapchainImageFoveationVulkanFB 作为 next 指针链
  存储到 CxVulkanOpenXrFoveationImageInfo
```

### 投影层缩放

```
new_width  = (recommended_width  * buffer_scale).max(1)
new_height = (recommended_height * buffer_scale).max(1)
```

`resize_projection_layer_vulkan` 在尺寸变化时创建新的交换链和渲染目标。旧资源保留在 `retired_projection_layers` 中，在 `destroy_session_vulkan` 时统一清理。这种"延迟销毁"确保当前正在渲染的帧不会引用已销毁的资源。

### 渲染流程 (`openxr_draw_pass_to_vulkan`)

```
1. 从 frame 获取 color_image_index 和 depth_image_index
2. 设置 pass uniforms (投影/视图/深度矩阵，双目)
3. 借用 self.os.vulkan (take/restore 模式)
4. vulkan.draw_openxr_view(self, draw_pass_id, draw_list_id, render_targets, color_index, depth_index)
5. [Android 可选] vulkan.read_openxr_color_image_rgba → Studio 帧
```

Vulkan 后端使用 `take/restore` 模式借用 — 渲染期间 `self.os.vulkan` 被临时取出，完成后放回。这允许 `draw_openxr_view` 拥有独占的可变访问。

### 销毁顺序

```
destroy_session_vulkan:
  1. 遍历退役投影层:
     - vulkan.destroy_openxr_session_data(render_targets)
     - xrDestroySwapchain(color_swap_chain)
  2. vulkan.destroy_openxr_session_data(current_render_targets)
  3. xr_tsdf_store().clear()
```
