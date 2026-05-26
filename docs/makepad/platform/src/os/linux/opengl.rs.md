# opengl.rs — OpenGL 渲染管线核心

**文件路径**: `platform/src/os/linux/opengl.rs` (3070 行)
**核心功能**: OpenGL ES 3.0 渲染管线的完整实现，包含 shader 编译、VAO 管理、纹理处理、渲染到纹理/窗口。

## 主要类型

### `CxOsDrawShader`
OS 级着色器表示：
- `gl_shader: [Option<GlShaderState>; 2]` — 窗口/XR 两个变体
- `vertex` / `pixel` — 窗口和 XR 的 GLSL 源码
- `live_uniforms` — 运行时 uniform 缓冲区
- （Vulkan 可选）`vulkan_shader`

### `GlShader`
编译链接后的 OpenGL 着色器程序：
- `program` — GL 程序 ID
- `geometries` / `instances` — 顶点属性和实例属性描述
- `textures` / `samplers` — 纹理和采样器
- `uniforms: GlShaderUniforms` — uniform 块绑定

### `GlShaderState`
着色器编译状态枚举：
- `Ready(GlShader)` — 已就绪
- `Pending(PendingGlShader)` — 异步编译中

### `GlShaderUniforms`
Uniform 块绑定集合：
- `pass_uniforms_binding` / `draw_list_uniforms_binding`
- `draw_call_uniforms_binding` / `user_uniforms_binding`
- `live_uniforms_binding` / `const_table_uniform`

### `CxOsGeometry` / `CxOsDrawCall` / `CxOsDrawCallVao` / `CxOsDrawList`
OS 层几何体、绘制调用、VAO 和绘制列表的 OpenGL 状态。

### `CxOsTexture` / `CxOsPass` / `CxOsUniformBuffer`
纹理、渲染通道和 uniform 缓冲区的 OpenGL 状态。

### `OpenglBuffer` / `OpenglUniform` / `OpenglSampler` / `OpenglUniformBlockBinding` / `OpenglAttribute`
OpenGL 资源包装类型。

### `EglRenderBridge`
EGL 渲染桥接，包装 EGL 上下文引用供外部使用（如 Servo 集成）。

## 关键方法

### `DrawVars::compile_shader(vm, apply, value)`
Makepad Shader 编译入口：
1. 检查三級缓存：对象 ID → shader → 函数哈希 → shader → 代码 → shader
2. 编译 Shader IR（`ShaderOutput`），后端为 `Glsl`
3. （可选）编译 Vulkan SPIR-V 版本
4. 生成 GLSL vertex/fragment shader 源码
5. 创建 `CxDrawShaderMapping`
6. 注册到 `draw_shaders` 缓存

### `Cx::render_view(draw_pass_id, draw_list_id, zbias, zbias_step)`
递归渲染视图树：
1. 更新 draw list uniforms
2. 遍历绘制顺序列表
3. 对每个子列表递归调用自身
4. 对每个绘制调用：
   - 确保 shader 已编译（同步或异步）
   - 更新 instance buffer、uniform 缓冲区
   - 管理 VAO（当 shader/buffer 变化时重建）
   - 绑定 uniforms、纹理、采样器
   - 调用 `glDrawElementsInstanced`

### `Cx::setup_render_pass(draw_pass_id)`
配置渲染通道：
- 计算视口矩阵
- 更新 pass uniforms

### `Cx::draw_pass_to_texture(draw_pass_id, override_pass_texture)`
渲染到帧缓冲对象（FBO）：
1. 创建/复用 FBO
2. 附加颜色纹理（支持 cube map face）
3. 附加深度模板缓冲区
4. 清除并渲染
5. 恢复默认帧缓冲

### `Cx::opengl_compile_shaders()`
编译所有待编译的 shader（异步编译轮询）。

### `Cx::create_gl_render_texture(width, height)`
创建可渲染纹理（供 Servo 集成使用）。

### CxTexture 方法
- `update_vec_texture` — 更新像素数据纹理（支持 BGRA、RGBAf32、R8 等格式，支持局部更新）
- `setup_video_texture` — 初始化视频外部纹理（OES 或标准 2D）
- `update_render_target` — 更新渲染目标纹理（支持 cube map）
- `update_depth_stencil` — 创建/更新深度模板渲染缓冲区
- `free_previous_resources` — 释放旧纹理资源

### `GlShader` 方法
- `new` — 完整编译：读取缓存 → 同步编译 → 构建
- `begin_state` — 尝试读取缓存，否则尝试并行编译
- `read_program_cache` / `write_program_cache` — 程序二进制缓存
- `opengl_get_attributes` / `opengl_get_texture_slots` / `opengl_create_samplers` — 反射属性
- `set_uniform_array` / `opengl_get_uniform` / `opengl_get_uniform_block_binding` — uniform 查询

### `CxOsDrawShader` 方法
- `new` — 生成含纹理扩展和 XR 支持的 GLSL 源码（`#version 300 es`）
- `ensure_gl_shader_sources` / `ensure_gl_shader_started` — 延迟编译
- `poll_gl_shader_ready` — 轮询异步编译完成状态

## Shader 生成

### 扩展处理
- 检测 `GL_OES_EGL_image_external` 支持
- Adreno GPU 禁用外部纹理（驱动 bug）
- Android 模拟器禁用外部纹理

### 版本与特性
- 基础 `#version 300 es`
- 支持 `GL_OVR_multiview2`（XR）
- 提供 `sample2d` / `samplecube` / `depth_clip` 等辅助函数

## 常量

- `SHADER_VARIANT_WINDOW = 0` / `SHADER_VARIANT_XR = 1`
- `NUM_SHADER_VARIANTS = 2`

## 调试功能

- `MAKEPAD_GL_DRAW_TRACE` — 记录 VAO 重建和绘制调用详情
- `MAKEPAD_DUMP_GLSL_IR` — 输出 GLSL 中间表示
- `MAKEPAD_LOG_GLSL_SOURCES` — 记录 GLSL 源码
