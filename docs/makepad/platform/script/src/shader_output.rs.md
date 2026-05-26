# shader_output.rs — ShaderOutput 特性与后端输出容器

**源码路径:** `platform/script/src/shader_output.rs`  
**总行数:** 627 行  
**核心职责:** 定义着色器编译输出的数据结构，包括 IO 列表、函数定义、结构体类型、采样器绑定、uniform 缓冲区索引分配等。作为编译器的**输出容器**，在编译过程中逐步填充，最终由后端代码生成器消费。

---

## 核心数据结构

### `ShaderSampler` — 采样器配置（第10-56行）

描述纹理采样器的参数：
- `filter`: `Nearest` 或 `Linear`（过滤模式）
- `address`: `Repeat`、`ClampToEdge`、`ClampToZero`、`MirroredRepeat`（寻址模式）
- `coord`: `Normalized` 或 `Pixel`（坐标系统）
- `is_video`: 是否为视频纹理

提供了 `Default` 实现（Linear + ClampToEdge + Normalized）和 `video()` 构造器。

### `ShaderStorageFlags` — 存储缓冲区标志（第61-79行）

位标志组合：第0位 = read，第1位 = write。提供 `set_read()`、`set_write()`、`is_read()`、`is_write()`、`is_readwrite()` 方法。

### `TextureType` — 纹理类型枚举（第81-94行）

- `Texture1d` / `Texture1dArray` / `Texture2d` / `Texture2dArray` / `Texture3d` / `Texture3dArray`
- `TextureCube` / `TextureCubeArray` / `TextureDepth` / `TextureDepthArray` / `TextureVideo`

### `ShaderIoKind` — Shader IO 种类（第96-110行）

描述 IO 数据的种类：
- `StorageBuffer(flags)`: 存储缓冲区
- `UniformBuffer`: uniform 缓冲区
- `Sampler(options)`: 采样器
- `Texture(type)`: 纹理
- `Varying`: 顶点→片元变量
- `VertexBuffer`: 顶点缓冲区
- `VertexPosition`: 顶点位置（`gl_Position` 等价物）
- `FragmentOutput(u8)`: 片元输出（多渲染目标索引）
- `RustInstance`: Rust 端实例字段（Repr(C) 对齐）
- `Uniform`: 动态 uniform
- `DynInstance`: 动态实例字段
- `ScopeUniform`: 作用域 uniform（来自脚本 scope 的非 IO 属性）

### `ScopeUniformSource` — 作用域 uniform 来源追踪（第112-126行）

记录每个 scope uniform 的来源：
- `source_obj`: 源对象
- `key`: 在源对象中的键
- `shader_name`: 着色器中使用的名称（可能前缀以避免冲突）
- `ty`: Pod 类型

用于编译后的缓冲区刷新——运行时需要知道从哪个对象读取哪些值来更新 uniform 缓冲区。

### `ScopeUniformBufferSource` — 作用域 uniform 缓冲区来源（第128-138行）

记录每个 scope uniform buffer 的信息（如 `let buf = shader.uniform_buffer(...)`）：
- `obj`: 缓冲区对象（用于运行时绑定）
- `pod_ty`: 缓冲区原型类型的 Pod 类型
- `shader_name`: 着色器中的名称

与 `ScopeUniformSource` 的区别在于后者是单个值，而这是整个缓冲区。

### `ScopeTextureSource` — 作用域纹理来源（第140-150行）

记录 scope 中定义的纹理：
- `obj`: 纹理对象（用于运行时绑定）
- `tex_type`: 纹理类型（2D、Cube 等）
- `shader_name`: 着色器中的名称

### `ShaderIo` — 单个 IO 条目（第152-178行）

代表一个注册的 shader IO 条目：
- `kind`: IO 种类
- `name`: 字段名称（LiveId）
- `ty`: Pod 类型
- `buffer_index`: 运行时分配的缓冲区索引（用于 uniform buffer 等）

提供 `kind()`、`name()`、`ty()`、`buffer_index()` 四个 getter 方法。

### `ShaderMode` — 着色器模式枚举（第180-186行）

- `Vertex`: 顶点着色器
- `Fragment`: 片元着色器（默认）
- `Compute`: 计算着色器

### `ShaderOutput` — 编译器输出容器（第188-212行）

编译器的主输出结构，在编译过程中被逐步填充。包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `mode` | `ShaderMode` | 当前编译模式 |
| `backend` | `ShaderBackend` | 目标后端（WGSL/Metal/HLSL/GLSL/Rust） |
| `use_vulkan` | `bool` | 是否使用 Vulkan（影响 GLSL 代码生成） |
| `io` | `Vec<ShaderIo>` | 所有已注册的 IO 条目 |
| `recur_block` | `Vec<ScriptObject>` | 递归检测栈（防止函数无限递归） |
| `structs` | `BTreeSet<ScriptPodType>` | 需要生成定义的结构体类型 |
| `functions` | `Vec<ShaderFn>` | 已编译的函数定义 |
| `samplers` | `Vec<ShaderSampler>` | 已注册的采样器 |
| `scope_uniforms` | `Vec<ScopeUniformSource>` | 作用域 uniform |
| `scope_uniform_buffers` | `Vec<ScopeUniformBufferSource>` | 作用域 uniform 缓冲区 |
| `scope_textures` | `Vec<ScopeTextureSource>` | 作用域纹理 |
| `texture_sampler_bindings` | `Vec<(String, usize)>` | 纹理→采样器绑定（GLSL 需要） |
| `hlsl_needs_tex_size` | `bool` | HLSL 是否需要 `_mpTexSize2D` 辅助函数 |
| `has_errors` | `bool` | 编译是否有错误 |
| `uses_derivatives` | `bool` | 是否使用了屏幕空间导数（dFdx/dFdy） |
| `rust_tmp_counter` | `usize` | Rust 后端临时变量 ID 计数器 |

### `UniformBufferBindings` — uniform 缓冲区索引映射（第214-231行）

存储 uniform 缓冲区类型名到缓冲区索引的映射：
- `bindings`: `Vec<(LiveId, usize)>` 类型名→索引对的列表
- `scope_uniform_buffer_index`: scope uniform 缓冲区的索引（如有）

提供 `get_by_type_name` 方法按类型名查询索引。

### `ShaderFn` — 已编译函数记录（第233-242行）

- `call_sig`: 函数声明签名（如 `fn foo(_iof: IoF, _iov: IoV, x: f32) -> f32`）
- `overload`: 重载编号（0 表示无重载）
- `name`: 函数名
- `args`: 参数 Pod 类型列表
- `fnobj`: 函数对象（用于重载检测）
- `out`: 函数体生成的代码字符串
- `ret`: 返回类型

---

## 方法详解

### `next_rust_tmp_id` — 获取 Rust 后端临时 ID（第245-249行）

单调递增（wrapping_add）的临时变量 ID 生成器，用于 Rust 后端表达式提权。

### `pre_collect_rust_instance_io` — 预收集 Rust 实例字段（第258-301行）

递归遍历 prototype 链，从最深的祖先到最深的派生类，收集所有 `RustInstance` 类型的字段。使用 `iter_rust_instance_ordered` 获取类型检查信息中的有序属性列表，通过 `type_id_to_pod_type` 获取 Pod 类型，注册到 `output.io`。

**重要性**：RustInstance 字段的布局必须与 Rust 端的 `#[repr(C)]` 结构体完全一致。先收集父类字段、再收集子类字段的顺序确保了正确的内存布局。DynInstance 字段不需要排序。

### `pre_collect_shader_io` — 预收集所有 Shader IO（第309-318行）

递归遍历 prototype 链，收集所有显式标记的 IO 字段（uniform、纹理、片段输出等）。同样使用"最深祖先优先"的顺序确保 IO 出现在定义顺序而非访问顺序中。然后为所有 `Uniform` 类型的 IO 字段设置 Pod 类型名称。

### `pre_collect_shader_io_recursive` — Shader IO 预收集递归实现（第321-409行）

在每个对象的 map 中迭代，检测 `shader_io` 标记。处理以下 IO 类型：
- **`SHADER_IO_DYN_UNIFORM`**（第347-356行）：注册为 `ShaderIoKind::Uniform`
- **`SHADER_IO_UNIFORM_BUFFER`**（第358-367行）：注册为 `ShaderIoKind::UniformBuffer`
- **`SHADER_IO_FRAGMENT_OUTPUT_0` ~ `SHADER_IO_FRAGMENT_OUTPUT_MAX`**（第369-387行）：按索引注册为 `FragmentOutput(index)`，避免按名称的重复
- **纹理类型**（第390-400行）：注册为对应的 `Texture` kind（不设置 Pod 类型，使用 `ScriptPodType::VOID`）

### `get_pod_type_from_value` — 从 ScriptValue 获取 Pod 类型（第411-437行）

尝试多种方式从值中提取 Pod 类型：检查是否为内置标量类型、颜色（→ `vec4f`）、pod 类型对象、pod 实例、pod 类型引用。

### `create_struct_defs` — 生成结构体定义字符串（第439-478行）

将 `self.structs` 中的 Pod 结构体类型分为两类：
- `plain_structs`: 普通结构体
- `packed_structs`: Metal 后端中需要 `packed_` 前缀的结构体（UniformBuffer、VertexBuffer、RustInstance、DynInstance）

在 Metal 后端调用 `pod_struct_defs_mixed`（分别对普通和 packed 结构体生成不同语法），其他后端调用 `pod_struct_defs`。

### `create_functions` — 生成已编译函数定义字符串（第480-496行）

遍历 `self.functions`，对每个函数输出 `call_sig { body }`。HLSL 后端特殊处理：当函数签名包含 `inout IoV _iov` 或 `inout IoF _iof` 时，在函数体首行加入 `_iov = _iov;` 或 `_iof = _iof;`，这是为了满足 DXC 编译器对 `inout` 参数的明确赋值要求。

### `find_vertex_buffer_object` — 查找顶点缓冲区对象（第499-529行）

在 `io_self` 的 prototype 链中查找标记为 `SHADER_IO_VERTEX_BUFFER` 的属性，返回其对象（用于顶点输入布局推导）。

### `assign_uniform_buffer_indices` — 分配 uniform 缓冲区索引（第534-565行）

遍历 `self.io`，为每个 `UniformBuffer` 分配递增的缓冲区索引。记录类型名→索引的映射到 `UniformBufferBindings`。如果存在 scope uniform，额外分配一个 `scope_uniform_buffer_index`（在最后一个 uniform buffer 索引之后）。返回 `(UniformBufferBindings, next_index)`。

### `get_uniform_buffer_bindings` — 从当前 IO 状态获取绑定映射（第569-600行）

在 `assign_uniform_buffer_indices` 已经调用后（`buffer_index` 已设置），从此方法获取当前绑定映射。scope uniform buffer 索引计算为 `max_buffer_index + 1`（默认 3）。

### `get_or_create_sampler` — 获取或创建采样器（第603-612行）

在 `self.samplers` 中查找匹配的采样器，如果不存在则创建新条目。返回采样器索引（该索引在后端被编码为采样器变量后缀 `_s{N}`）。

### `bind_texture_sampler` — 绑定纹理到采样器（第614-626行）

记录纹理表达式和采样器索引的对应关系。如果同一纹理已被绑定过，保留第一个（忽略后续绑定）。此信息供 GLSL 后端使用——GLSL 需要生成纹理→采样器绑定辅助函数。
