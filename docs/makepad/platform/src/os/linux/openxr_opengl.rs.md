# openxr_opengl.rs

One-liner (EN): OpenXR session creation and rendering for the OpenGL ES backend — multiview framebuffer setup, depth texture hook, stereo projection layer rendering.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/openxr_opengl.rs` (358 行)
- **核心作用**: 实现 OpenXR 的 OpenGL ES 后端：创建 XR 会话（绑定 EGL 上下文）、创建和配置立体交换链（multiview array texture）、管理深度纹理和帧缓冲对象、执行双目立体渲染。

## 关键方法

### Cx 方法

| 方法 | 说明 |
|------|------|
| `openxr_draw_pass_to_multiview(draw_pass_id, frame)` | 渲染 OpenXR 立体 pass：设置双目的投影/视图矩阵到 uniform → 绑定交换链帧缓冲 → [Android] 可选的 Studio 帧读取 → 清除 → 渲染视图列表 |

### CxOpenXr 方法

| 方法 | 说明 |
|------|------|
| `depth_texture_hook(gl, shgl, mapping)` | 深度纹理挂钩：将环境深度交换链的 `TEXTURE_2D_ARRAY` 绑定到着色器的 `xr_depth_texture` uniform |

### CxOpenXrSession 方法

| 方法 | 说明 |
|------|------|
| `create_session_gles(xr, system_id, instance, display, options) -> Result<Self, String>` | **创建 OpenGL ES XR 会话**：创建 session → 描述立体配置 → 创建颜色交换链 → 枚举交换链图像 → 创建透传和深度提供器 → 创建深度交换链纹理 → 为每个交换链图像创建 GL framebuffer 对象（含 depth attachment）→ 启动深度提供器 → 创建 inputs |
| `destroy_session_gles(gl)` | 销毁 GL 资源：删除所有深度纹理和帧缓冲对象 |

## 实现细节

### 立体渲染流程 (`openxr_draw_pass_to_multiview`)

```
frame.swap_chain_index → 当前交换链图像

1. 设置 pass uniform:
   - camera_projection/view (左目)
   - camera_projection_r/view_r (右目)
   - camera_inv/inv_r (逆视图)
   - depth_projection/view (深度)

2. 绑定帧缓冲 → gl_frame_buffers[swap_chain_index]

3. 设置 OpenGL 状态:
   - glBindFramebuffer(DRAW_FRAMEBUFFER, fb)
   - glColorMask/DEPTH_MASK = TRUE
   - glEnable(SCISSOR_TEST, DEPTH_TEST, BLEND)
   - glDepthFunc(LEQUAL)
   - glBlendFuncSeparate(ONE, ONE_MINUS_SRC_ALPHA)
   - glViewport/Scissor(0, 0, width, height)
   - glClearColor(0,0,0,0), glClearDepthf(1.0)

4. render_view() → 执行实际绘制

5. [Android 可选] glReadPixels → 编码 Studio 帧

6. glBindFramebuffer(0) → 解绑
```

### 交换链配置

| 参数 | 值 |
|------|-----|
| Usage flags | `SAMPLED | COLOR_ATTACHMENT` |
| 格式 | `SRGB8_ALPHA8` (GL) |
| Array 大小 | 2（双目） |
| 采样数 | 1（或 `options.multisamples`） |

### GL 帧缓冲创建

每个交换链图像创建对应的 GL 帧缓冲：
```
for each swapchain_image:
  1. 配置颜色纹理 (TEXTURE_2D_ARRAY):
     - WRAP: CLAMP_TO_BORDER
     - FILTER: LINEAR
     - 边界颜色: [0,0,0,0]
  2. 创建深度纹理 (DEPTH_COMPONENT24)
  3. 创建帧缓冲对象
  4. 绑定颜色纹理到 COLOR_ATTACHMENT0
  5. 绑定深度纹理到 DEPTH_ATTACHMENT
  6. 若 multisamples>1 → FramebufferTextureMultisampleMultiviewOVR
     否则 → FramebufferTextureMultiviewOVR
```

使用 OVR_multiview 扩展的 `FramebufferTextureMultiviewOVR`（或 MSAA 版本的 `MultisampleMultiviewOVR`），层范围设置为 `[0, 2)`。

### 深度纹理挂钩

XR 着色器的深度采样器绑定到环境深度交换链当前索引的 `TEXTURE_2D_ARRAY`，使用 `NEAREST` 过滤和 `CLAMP_TO_EDGE` 包裹模式。

### Android 兼容性

- 文件内条件编译 `#[cfg(target_os = "android")]` 用于 Studio 帧读取功能
- 使用 `XrGraphicsBindingOpenGLESAndroidKHR` 作为图形绑定（包含 EGL display/config/context）
