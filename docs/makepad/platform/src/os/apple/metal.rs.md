# metal.rs — Metal 图形渲染后端

**文件路径:** `platform/src/os/apple/metal.rs`

**核心目的:** 实现 Makepad 框架的 Metal 图形渲染后端。提供 Metal 设备管理、着色器编译、渲染管道状态、纹理管理、缓冲区管理和绘制调用封装。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `MetalBackend` | Metal 渲染后端主结构体，实现 `DrawShaderTrait` |
| `MetalDevice` | Metal 设备封装（MTLDevice 的包装） |
| `MetalQueue` | 命令队列封装（MTLCommandQueue） |
| `MetalPipeline` | 渲染管道状态封装（MTLRenderPipelineState） |
| `MetalShader` | 着色器封装（MTLFunction + 反射信息） |
| `MetalBuffer` | 缓冲区封装（MTLBuffer + 偏移管理） |
| `MetalTexture` | 纹理封装（MTLTexture + 视图） |
| `MetalDrawQuad` | 四边形绘制实例数据 |
| `MetalUniformBuffer` | Uniform 常量缓冲区 |
| `MetalVertexBuffer` | 顶点缓冲区 |
| `MetalIndexBuffer` | 索引缓冲区 |
| `MetalDepthStencilState` | 深度/模板状态 |
| `MetalSamplerState` | 采样器状态 |
| `MetalCommandBuffer` | 命令缓冲区封装 |
| `MetalRenderPass` | 渲染通道封装 |
| `MetalSwapchain` | 交换链管理（CAMetalLayer 驱动） |

**`MetalBackend` 关键方法:**
- `MetalBackend::new(cx)` — 初始化 Metal 后端：
  - 创建 MTLDevice（`MTLCreateSystemDefaultDevice`）
  - 创建 MTLCommandQueue
  - 初始化默认着色器库
  - 设置 Metal 调试层（可选）
- `MetalBackend::begin_frame(drawcall)` — 开始新帧渲染
- `MetalBackend::end_frame()` — 结束帧渲染并提交命令缓冲区
- `MetalBackend::present(drawable)` — 呈现 drawable
- `MetalBackend::compile_shader(source, entry, kind)` — 编译 Metal 着色器函数
- `MetalBackend::create_pipeline(desc)` — 创建渲染管道状态
- `MetalBackend::create_buffer(size, usage)` — 创建 GPU 缓冲区
- `MetalBackend::create_texture(desc)` — 创建 GPU 纹理
- `MetalBackend::draw_quad(...)` — 绘制一个四边形

**着色器管理:**
- 支持 Metal Shading Language (MSL) 着色器
- 静态库中嵌入预编译着色器（通过 `air` 汇编）
- 运行时编译支持（`newLibraryWithSource`）
- 着色器反射：自动解析参数缓冲区绑定和结构体布局
- 支持 Metal 2.0+ 功能集

**渲染管道:**
- 图形管道描述符配置：
  - 顶点和片段着色器
  - 混合状态（支持预乘 alpha、源叠加）
  - 深度/模板测试
  - 光栅化（填充模式、剔除模式、线宽）
  - 顶点描述（布局、步幅、属性格式）
- 管道缓存：按描述符哈希缓存已编译的管道状态

**交换链:**
- 使用 `CAMetalLayer` 作为可绘制呈现目标
- `CAMetalLayer` 管理：
  - 像素格式（BGRA8Unorm 或 BGRA8Unorm_sRGB）
  - 帧尺寸和缩放因子（支持 Retina）
  - 绘制顺序（`MTLPixelFormat`）
  - 帧间隔（vsync 配置）
- `nextDrawable` 获取下一个可绘制纹理
- 呈现通过 `presentDrawable:` 或 `presentDrawable:afterMinimumDuration:`
- 支持 Display P3 广色域

**纹理管理:**
- 支持多种纹理类型：2D、2D Array、Cubemap
- 像素格式覆盖：
  - RGBA8Unorm、BGRA8Unorm（标准）
  - R8Unorm（单通道，用于 Y 平面）
  - RG8Unorm（双通道，用于 UV 平面）
  - Float16、Float32（HDR）
  - SRGB 变体
- 纹理创建：从图片数据、原始像素数据、CVPixelBuffer
- 纹理上传通过 `replaceRegion:mipmapLevel:withBytes:bytesPerRow:`
- Mipmap 自动生成（`generateMipmapsForTexture:`）

**缓冲区管理:**
- 统一缓冲区（Uniform Buffer）：用于每帧常量数据
- 顶点缓冲区：动态更新每帧顶点数据
- 索引缓冲区：支持 16 位和 32 位索引
- 使用共享内存模式（`MTLResourceStorageModeShared`）或私有内存 + blit 复制
- 缓冲区循环分配（ring buffer）以避免 GPU 与 CPU 同步等待
- 对齐要求：缓冲区偏移必须满足设备对齐限制

**绘制调用：四边形（Quad）批处理系统**
1. 四边形实例数据打包到实例缓冲区
2. 着色器通过 instance_id 访问实例属性
3. 统一缓冲区包含视图投影矩阵和全局参数
4. 可见四边形通过裁剪剔除优化
5. 支持纹理图集（texture atlas）

**命令缓冲区和编码器:**
- `MTLCommandBuffer` 管理：
  - 命令编码器（`MTLRenderCommandEncoder`, `MTLBlitCommandEncoder`）
  - 提交和等待完成
  - GPU 时间戳捕获
- `MTLRenderCommandEncoder` 配置：
  - 设置渲染管道状态
  - 设置顶点/片段缓冲区
  - 设置纹理和采样器
  - 设置视口和裁剪矩形
  - 绘制基元（三角形、线条）

**GPU 时间线和同步:**
- `MTLFence` / `MTLSharedEvent` 用于跨命令缓冲区同步
- `waitUntilCompleted` 阻塞等待 GPU 完成
- 信号量用于 CPU-GPU 同步（最大缓冲帧数 = 3）
- `MTLHeap` 用于高效内存分配

**调试和性能:**
- Metal GPU 捕获支持（通过 `MTLCaptureManager`）
- 调试组标签（`pushDebugGroup` / `popDebugGroup`）
- 计数器集（`MTLCounterSet`）
- 性能统计

**平台集成:** macOS、iOS、tvOS，使用 Metal 框架
