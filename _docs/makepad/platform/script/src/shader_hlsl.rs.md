# `shader_hlsl.rs` — DirectX HLSL 后端代码生成器

## 概述

本文件实现了 Makepad 着色器编译器的 DirectX HLSL 后端，将中间表示翻译为 HLSL（Shader Model 5.0+）着色器源码。与 GLSL 后端不同，HLSL 后端采用**结构体驱动**的代码生成策略：创建显式的 `struct` 声明来描述顶点输入、varying、实例数据和 uniform 布局，使用全局静态变量模拟统一的可变状态访问路径。

**核心特点：**
- 使用 `cbuffer` 声明 uniform 常量缓冲，并使用 `register(bN)` 显式指定槽位
- 通过语义（`SV_POSITION`、`SV_TARGETN`、`SV_VertexID`、`SV_InstanceID`）绑定系统值
- 使用自定义语义 `GEOM`/`INST`/`VARY` 进行顶点输入元素和 varying 的语义化绑定
- 静态全局变量 `_mp_io`、`_mp_iov`、`_mp_iof` 模拟统一上下文
- 矩阵采用转置（`transpose`）写入以匹配 HLSL 的行优先构造函数

---

## 私有方法

### `hlsl_is_integer_like_input(vm: &ScriptVm, ty: ScriptPodType) -> bool`

判断给定的 Pod 类型是否需要 `nointerpolation` 限定符。如果类型是 `U32`/`I32`/`Bool`/`AtomicU32`/`AtomicI32` 及其向量形式（`Vec2u`/`Vec3u`/`Vec4u`/`Vec2i`/`Vec3i`/`Vec4i`/`Vec2b`/`Vec3b`/`Vec4b`），返回 `true`。这些类型在插值阶段会产生未定义行为，必须标记为 `nointerpolation`。

### `hlsl_slot_chunks(slots: usize) -> Vec<usize>`

将 slot 数量分块以适应 HLSL 顶点输入语义的 4 分量限制。

- `0` slots：返回空切片
- `9` slots（`Mat3x3f`）：分成 `[3, 3, 3]` 三个三分量向量
- `16` slots（`Mat4x4f`）：分成 `[4, 4, 4, 4]` 四个四分量向量
- 其他：每次最多取 4，直到取完（如 5→[4, 1]）

### `hlsl_chunk_ty(slots: usize) -> &'static str`

根据 chunk 的大小（1-4）返回对应的 HLSL 类型名：`float`/`float2`/`float3`/`float4`。

### `hlsl_input_needs_chunks(vm: &ScriptVm, ty: ScriptPodType) -> bool`

判断某个类型是否需要被分块输入。条件：slot 数大于 4（如大于 `vec4` 的类型）或者该类型是一个结构体。结构体需要分块是因为 HLSL 不支持直接在顶点声明中嵌套用户定义的结构体作为输入语义。

### `hlsl_reconstruct_from_scalars(&self, vm, ty, scalars, scalar_idx) -> String`

从标量列表递归重构 HLSL 表达式。对于结构体类型，生成 `consfn_{StructName}(field1, field2, ...)` 形式的构造函数调用。对于非结构体：

- 1 slot 以下：直接取标量值
- 向量/矩阵：用 `Type(args)` 形式重构
- 矩阵特殊处理：CPU 端存储为列优先，而 HLSL 构造函数要求行优先输入，因此包裹 `transpose(Type(args))`

### `hlsl_reconstruct_input_value(&self, vm, ty, input_prefix, io_name) -> String`

从 HLSL 顶点输入结构体重建某个 IO 的完整值：

1. 如果类型不需要分块，直接从 `input.{prefix}_{io_name}` 读取
2. 如果需要分块，逐个读取 `input.{prefix}_{io_name}_{chunk_idx}` 的分量
3. 收集所有标量分量后调用 `hlsl_reconstruct_from_scalars` 重构为完整类型

---

## 公有方法

### `hlsl_create_helpers(&self, _vm, out)`

生成 HLSL 辅助函数。目前仅包含 `_mpTexSize2D`，它使用 HLSL 的 `GetDimensions` 方法获取纹理尺寸。仅当着色器使用了 `tex_size` 操作时才会生成。

### `hlsl_create_instance_struct(&self, vm, out)`

生成 `IoInstance` 结构体，包含所有动态实例（`DynInstance`）和 Rust 实例（`RustInstance`）字段。动态实例字段在前，Rust 实例字段在后。这个顺序与 `pre_collect_rust_instance_io` 的执行次序配合，确保偏移量与 CPU 侧保持一致。

### `hlsl_create_uniform_struct(&self, vm, out)`

生成 `cbuffer IoUniform : register(b2)` 常量缓冲。包含所有标记为 `Uniform` 的 IO 字段。固定绑定在 slot `b2`，与 Makepad 的绘制管线约定一致。

### `hlsl_create_scope_uniform_cbuffer(&self, vm, out)`

生成 `cbuffer IoScopeUniform` 常量缓冲，包含所有 `ScopeUniform` 字段。只有在存在 scope uniform 时才生成。buffer index 通过查找当前 IO 中最大的 `buffer_index` 加 1 来确定，避免与 UniformBuffer 冲突。

### `hlsl_create_uniform_buffer_cbuffers(&self, vm, out)`

为每个 `UniformBuffer` 类型的 IO 生成独立的 `cbuffer` 声明。每个 cbuffer 通过 `register(bN)` 绑定到预分配的 buffer index：

```hlsl
cbuffer cb_passUniforms : register(b3) { DrawPassUniforms u_draw_pass; };
```

### `hlsl_create_varying_struct(&self, vm, out)`

生成 `IoVarying` 结构体，包含需要在顶点和片元之间传递的所有数据：

1. 动态实例、Rust 实例和 Varying 字段
2. 整数类型字段带 `nointerpolation` 限定符
3. 每个字段分配 `VARY{Letter}` 语义（从 `TEXCOORD0` 开始映射）
4. `_iid`（实例 ID）使用 `nointerpolation uint TEXCOORD0`
5. `_position` 使用 `float4 SV_POSITION`

### `hlsl_create_vertex_buffer_struct(&self, vm, out)`

生成 `IoVertexBuffer` 结构体，包含所有 `VertexBuffer` 类型的字段。

### `hlsl_create_vertex_input_struct(&self, vm, out)`

生成 `VertexInput` 结构体，这是顶点 shader 的完整输入布局描述：

1. **顶点缓冲字段** — 每个字段分配 `GEOM{Letter}{chunk_idx}` 语义。需要分块的字段（如 Matrix）展开为多个 float2/float3/float4 输入
2. **动态实例字段** — 分配 `INST{Letter}{chunk_idx}` 语义
3. **Rust 实例字段** — 同样分配 `INST` 语义，语义索引在动态实例之后连续分配
4. `vid : SV_VertexID` — 顶点索引
5. `iid : SV_InstanceID` — 实例索引

### `hlsl_create_io_structs(&self, vm, out)`

生成三个全局结构体和三个静态全局变量：

- `IoV` — 顶点上下文，包含 `v`（varying）、`vb`（顶点缓冲）、`i`（实例数据）、`vid`、`iid`
- `IoF` — 片元上下文，包含 `v`（varying）和每个片元输出 `fbN`
- `Io` — 空结构体（扩展预留）
- 静态变量 `_mp_io`/`_mp_iov`/`_mp_iof` — 全局上下文，用于在自定义着色器函数中访问输入/输出

### `hlsl_create_fragment_output_struct(&self, vm, out)`

生成 `IoFb` 结构体，包含所有片元输出字段。每个字段绑定 `SV_TARGET{index}` 语义用于 MRT。

### `hlsl_create_texture_samplers(&self, _vm, out)`

生成纹理和采样器声明：

1. 为每个纹理 IO 生成 `TextureX name : register(t{idx})`（`t` 寄存器空间）
2. 为每个采样器 IO 生成 `SamplerState name : register(s{idx})`（`s` 寄存器空间）
3. 为 `self.samplers` 中的预设采样器生成状态对象声明，包含：
   - `Filter = MIN_MAG_MIP_POINT` 或 `MIN_MAG_MIP_LINEAR`
   - `AddressU/V/W = Wrap` / `Clamp` / `Border` / `Mirror`

纹理类型到 HLSL 类型的映射：

| TextureType | HLSL 类型 |
|------------|-----------|
| Texture1d | Texture1D |
| Texture1dArray | Texture1DArray |
| Texture2d | Texture2D |
| Texture2dArray | Texture2DArray |
| Texture3d | Texture3D |
| Texture3dArray | Texture3D（不支持 true 3D array） |
| TextureCube | TextureCube |
| TextureCubeArray | TextureCubeArray |
| TextureDepth | Texture2D |
| TextureDepthArray | Texture2DArray |
| TextureVideo | Texture2D |

### `hlsl_create_vertex_fn(&self, vm, out)`

生成 `vertex_main` 函数，HLSL 顶点着色器的入口点：

1. 将 `VertexID` 和 `InstanceID` 写入 `_mp_iov`
2. 将每个顶点缓冲字段从 `input` 拷贝到 `_mp_iov.vb`
3. 将动态实例和 Rust 实例字段从 `input` 拷贝到 `_mp_iov.i` 和 `_mp_iov.v`
4. 调用 `io_vertex` 函数（如果返回值是 `Vec4f` 则赋给 `_position`）
5. 确保 `_iid` 被更新
6. 返回 `_mp_iov.v`（即 `IoVarying` 结构体）供光栅化阶段使用

### `hlsl_create_fragment_fn(&self, _vm, out)`

生成 `pixel_main` 函数，HLSL 片元着色器的入口点：

1. 将输入的 `IoVarying` 赋值到 `_mp_iof.v`
2. 调用 `io_fragment` 函数
3. 从 `_mp_iof.fbN` 收集所有片元输出到 `IoFb` 结构体
4. 返回 `IoFb` 结构体用于 MRT 渲染

### `index_to_semantic(index: usize) -> String`（独立函数）

将数字索引映射为 HLSL 语义后缀，使用 26 进制（A-Z）表示：

- 0 → A, 1 → B, ..., 25 → Z
- 26 → AA, 27 → AB, ..., 51 → AZ
- 52 → BA, ...

这种编码确保每个顶点输入元素和 varying 都有唯一的语义名称，支持超过 26 个属性的场景。
