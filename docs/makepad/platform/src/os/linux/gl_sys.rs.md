# gl_sys.rs — OpenGL ES FFI 绑定

**文件路径**: `platform/src/os/linux/gl_sys.rs` (822 行)
**核心功能**: OpenGL ES 3.0 API 的 FFI 绑定和动态加载，包含所有函数指针类型、枚举常量和 `LibGl` 结构体。

## OpenGL 常量

定义了 80+ 个 OpenGL 枚举常量，包括：
- Buffer 类型：`ARRAY_BUFFER` / `ELEMENT_ARRAY_BUFFER` / `UNIFORM_BUFFER`
- 纹理类型：`TEXTURE_2D` / `TEXTURE_CUBE_MAP` / `TEXTURE_EXTERNAL_OES`
- 格式：`RGBA` / `BGRA` / `RGBA32F` / `R8` / `R32F` / `SRGB8_ALPHA8L`
- 滤镜：`LINEAR` / `NEAREST` / `LINEAR_MIPMAP_LINEAR`
- 包装：`CLAMP_TO_EDGE` / `REPEAT` / `MIRRORED_REPEAT`
- 帧缓冲：`FRAMEBUFFER` / `COLOR_ATTACHMENT0` / `DEPTH_ATTACHMENT`
- Shader：`VERTEX_SHADER` / `FRAGMENT_SHADER` / `COMPILE_STATUS` / `LINK_STATUS`

## 函数指针类型

定义了 50+ 个 `Tgl*` 类型别名，覆盖：
- VAO管理：`TglGenVertexArrays` / `TglBindVertexArray` / `TglDeleteVertexArrays`
- Buffer管理：`TglGenBuffers` / `TglBindBuffer` / `TglBufferData`
- Shader编译：`TglCreateShader` / `TglShaderSource` / `TglCompileShader` / `TglCreateProgram`
- 纹理操作：`TglGenTextures` / `TglTexImage2D` / `TglTexSubImage2D`
- 渲染：`TglDrawElementsInstanced` / `TglDrawArrays`
- 帧缓冲：`TglGenFramebuffers` / `TglFramebufferTexture2D`
- 调试：`TglDebugMessageCallback` / `TglDebugMessageControl`
- 多视图：`TglFramebufferTextureMultiviewOVR`
- 平行编译：`TglMaxShaderCompilerThreadsKHR`

## `LibGl` 结构体

包含所有 OpenGL 函数指针，通过 `try_load` 方法动态加载。每个函数有多个后备名称（如 `glGenVertexArrays` / `glGenVertexArraysAPPLE` / `glGenVertexArraysOES`）。

### `LibGl::enable_debugging()`
启用 OpenGL 调试输出，注册调试回调函数 `debug` 并激活所有调试消息。

### `LibGl::try_load(loadfn)`
通用加载方法，接受闭包作为符号解析器。使用 `load!` 宏按优先级顺序尝试多个符号名。

## 辅助宏

- `gl_log_error!` — 循环调用 `glGetError` 并记录所有非零错误
- `gl_flush_error!` — 静默清空 OpenGL 错误队列
