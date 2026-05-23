# `shader_wgsl.rs` — WebGPU WGSL 后端代码生成器

## 概述

本文件实现了 Makepad 着色器编译器的 WebGPU WGSL 后端，是四个后端中代码量最大（1118 行）的实现。它将中间表示翻译为 WGSL（WebGPU Shading Language）着色器源码，用于在 Vulkan/Metal/DirectX 12 的统一抽象层上运行。

**核心特点：**
- 使用 `@group(0) @binding(N)` 语法统一管理所有资源绑定
- 私有全局变量（`var<private>`) 模拟统一访问路径，各着色器阶段在入口函数中从 packed 输入/资源中装载
- 使用 `struct` 描述顶点输入 `VertexMainIn`、顶点输出 `VertexMainOut`、片元输入 `FragmentMainIn` 和片元输出 `FragmentMainOut`
- 内置深度裁剪（`depth_clip`）和 XR 多视图（`xr_multiview`）支持
- 通过 `bitcast<T>` 实现位编码整数类型在 vec4f 打包中的精确恢复

**核心数据结构：**
- `WgslDrawShaderSource` — 编译结果，包含 WGSL 源码和各绑定点索引
- `WgslPackedFormat` — 打包格式（Float/UInt/SInt）
- `WgslPackedField` — 打包字段元数据（名称、类型、slot 数、偏移量）

---

## 私有函数

### `wgsl_texture_type(tex_type: TextureType) -> &'static str`

纹理类型到 WGSL 类型的映射。注意 WGSL 的命名规范为全小写下划线分隔：

| TextureType | WGSL 类型 |
|------------|-----------|
| Texture1d | texture_2d\<f32\> |
| Texture1dArray | texture_2d_array\<f32\> |
| Texture2d | texture_2d\<f32\> |
| Texture2dArray | texture_2d_array\<f32\> |
| Texture3d | texture_3d\<f32\> |
| Texture3dArray | texture_3d\<f32\>（不支持 true 3D array） |
| TextureCube | texture_cube\<f32\> |
| TextureCubeArray | texture_cube_array\<f32\> |
| TextureDepth | texture_depth_2d |
| TextureDepthArray | texture_depth_2d_array |
| TextureVideo | texture_2d\<f32\> |

`Texture1d` 映射为 `texture_2d` 是 WGSL 的限制（不原生支持 1D 纹理）。

### `wgsl_type_name(output, vm, ty) -> String`

通过 backend 的 `pod_type_name_from_ty` 获取某 Pod 类型的 WGSL 名称。

### `wgsl_type_name_inline(output, vm, ty) -> String`

获取内联类型的 WGSL 名称。命名的结构体使用 `map_pod_name` 映射，匿名结构体使用 `S{index}` 格式，其他类型委托给 `backend.pod_type_name`。

### `wgsl_attr_format_from_pod_ty(ty) -> WgslPackedFormat`

与 GLSL 后端的 `glsl_attr_format_from_pod_ty` 逻辑相同：无符号/布尔类型→UInt，有符号→SInt，浮点→Float。

### `wgsl_num_packed_vec4s(slots) -> usize`, `wgsl_swizzle_component`, `wgsl_packed_component`

与 GLSL 后端相同的 slot→vec4 分块计算、swizzle 映射和打包表达式生成工具函数。

### `wgsl_push_field(output, vm, io, prefix, attribute_packing, offset, out)`

核心打包逻辑，与 GLSL 后端的 `glsl_push_field` 逻辑完全一致：计算字段需求 slot 数，在非 Float 格式的多 slot 字段前后进行 4 对齐。

### `wgsl_collect_geometry_fields(output, vm) -> Vec<WgslPackedField>`

收集所有 `VertexBuffer` 类型的 IO，调用 `wgsl_push_field` 构建几何体打包字段列表。

### `wgsl_collect_instance_fields(output, vm) -> Vec<WgslPackedField>`

收集所有 `DynInstance` 和 `RustInstance` 类型的 IO，使用属性打包模式。

### `wgsl_collect_varying_fields(output, vm) -> Vec<WgslPackedField>`

收集所有需要传递到片元阶段的字段（`DynInstance`、`RustInstance`、`Varying`），使用非属性打包模式（NumericFloat 编码）。

### `wgsl_to_float_scalar_expr(ty, expr) -> String`

标量类型到浮点表达式的转换：

- `F32`/`F16`：原样
- `U32`/`I32`/`AtomicU32`/`AtomicI32`：`f32(expr)`
- `Bool`：`select(0.0, 1.0, expr)`

使用 WGSL 内置 `select` 代替 GLSL 的三元运算符，因为 WGSL 不直接支持布尔到浮点的隐式转换。

### `wgsl_convert_scalar_expr(source, target, expr) -> String`

根据打包源编码方式转换标量表达式。关键差异：

- **NumericFloat 源**：`f32(expr)`→`f16(expr)`、`u32(expr)`、`i32(expr)`
- **BitPackedFloat 源**：使用 WGSL 的 `bitcast<u32>(expr)`/`bitcast<i32>(expr)` 而非 GLSL 的 `floatBitsToUint/floatBitsToInt`

### `wgsl_take_scalar_or_zero(scalars, scalar_index) -> String`

与 GLSL 后端相同，从标量列表中取值，越界返回 `"0.0"`。

### `wgsl_flatten_inline(output, ty, expr, out)`

递归展平复合类型表达式为标量分量。

- **Struct**：`(expr).field_name`
- **Vec**：`(expr).x/y/z/w`
- **Mat**：`(expr)[col][row]`

WGSL 中矩阵访问使用 `[][]` 语法而非 GLSL 的 `[][][]`。

### `wgsl_flatten_exprs(output, vm, ty, expr, out)`

类型化展平入口，包装后委托给 `wgsl_flatten_inline`。

### `wgsl_reconstruct_inline(output, vm, ty, source, scalars, scalar_index) -> String`

递归从标量列表重建复合类型表达式。与 GLSL 后端的 `glsl_reconstruct_inline` 逻辑完全相同：Struct→构造函数表达式、Vec→`vecNf(c1, c2, ...)`、Mat→`matNxNf(c1, c2, ...)`、Scalar→直接取值。WGSL 的向量/矩阵构造函数与 GLSL 风格兼容。

### `wgsl_unpack_expr_for_field(output, vm, field, prefix, source) -> String`

为打包字段生成解包表达式。收集字段所有 slot 对应的分量引用后调用 `wgsl_reconstruct_inline` 重构。

---

## 核心代码生成

### `build_draw_shader_wgsl(vm, output, xr_multiview) -> (String, u32, u32, u32, u32)`

这是 WGSL 后端的核心函数，返回 WGSL 源码字符串和四个绑定元数据（dynamic uniform binding、texture binding base、sampler binding base、XR depth binding）。整个函数按以下顺序构造 WGSL 源码：

#### 1. 准备工作

- 收集几何体、实例和 varying 的打包字段，计算 varying slots
- 检测是否存在动态 uniform 和作用域 uniform
- 计算所有资源绑定点：dynamic uniform 固定在 binding(2)，uniform buffer 使用预分配索引，作用域 uniform 在 uniform buffer 之后
- 计算纹理和采样器的起始绑定点

#### 2. 结构体定义

调用 `output.create_struct_defs(vm, &mut out)` 生成所有 Pod 类型的 WGSL 结构体声明。

#### 3. Uniform 声明

- 如果有动态 uniform：生成 `MpDynUniforms` 结构体和 `@group(0) @binding(2) var<uniform> _mp_dyn_uniforms`
- 如果有作用域 uniform：生成 `MpScopeUniforms` 结构体和对应的绑定点
- 声明 `VIEW_ID`（XR 多视图下的视图索引）和 `vtx_pos`（顶点位置）为 `var<private>`

#### 4. 全局变量声明

为每个 IO 类型生成 `var<private>` 全局变量：

| IO 类型 | 前缀 | 声明方式 |
|---------|------|---------|
| VertexBuffer | `vb_` | `var<private>` |
| DynInstance | `dyninst_` | `var<private>` |
| RustInstance | `rustinst_` | `var<private>` |
| Varying | `var_` | `var<private>` |
| Uniform | `uni_` | `var<private>` |
| UniformBuffer | `unibuf_` | `@group(0) @binding(N) var<uniform>` |
| ScopeUniform | `su_` | `var<private>` |
| FragmentOutput | `frag_fbN` | `var<private>` |
| StorageBuffer | `sb_` | `@group(0) @binding(N) var<storage, read_write>` |
| Texture | `tex_` | `@group(0) @binding(N) var tex_X: texture_xx` |
| Sampler | `sampler_` | `@group(0) @binding(N) var sampler_X: sampler` |

UniformBuffer 使用预分配 buffer index 作为 `@binding` 值。纹理和采样器使用动态分配的 `next_binding` 计数器。

#### 5. 预设采样器

遍历 `output.samplers` 生成 `_sN` 采样器变量的 WGSL 声明，也与上述绑定计数器相连。

#### 6. XR 深度纹理

在下一个绑定点生成 `tex_xr_depth` 声明。根据 `xr_multiview` 参数选择 `texture_depth_2d` 或 `texture_depth_2d_array`。

#### 7. 输入/输出结构体

**`VertexMainIn`** — 顶点着色器输入结构：
- `@builtin(view_index)` — 仅在 `xr_multiview` 时生成
- `@location(N) packed_geometry_N: vec4f` — 每个几何体 vec4 分块
- `@location(N) packed_instance_N: vec4f` — 每个实例 vec4 分块（紧接几何体之后）

**`VertexMainOut`** — 顶点着色器输出结构：
- `@builtin(position) position: vec4f`
- `@location(N) packed_varying_N: vec4f` — 每个 varying vec4 分块

**`FragmentMainIn`** — 片元着色器输入结构：
- `@builtin(view_index)` — 仅在 `xr_multiview` 时生成
- `@builtin(position) position: vec4f`
- `@location(N) packed_varying_N: vec4f`

**`FragmentMainOut`** — 片元着色器输出结构（仅在有片元输出时生成）：
- `@location(N) fbN: type` — 每个片元输出

#### 8. draw_pass 矩阵包装函数

生成五个包装函数，用于支持 XR 多视图下的左右眼矩阵选择：

- `mp_draw_pass_camera_projection()` — 根据 `VIEW_ID` 返回 `camera_projection` 或 `camera_projection_r`
- `mp_draw_pass_camera_view()` — 同理
- `mp_draw_pass_depth_projection()` — 同理
- `mp_draw_pass_depth_view()` — 同理
- `mp_draw_pass_camera_inv()` — 同理

这些函数与 GLSL 后端的 `draw_pass` uniform block 特殊处理（`name[2]` 数组）对应。

#### 9. 用户函数

调用 `output.create_functions(&mut out)` 嵌入编译后的着色器函数定义。

#### 10. `depth_clip` 函数

生成 `depth_clip(world, color, clip) -> vec4f` 函数，实现深度写入/裁剪：

1. 如果 `clip < 0.5`，跳过深度写入，直接返回颜色
2. 获取 depth projection/view 矩阵
3. 检查 depth_view 是否为有效深度矩阵（`abs(depth_view[3].w - 1.0) > 0.5` 时跳过）
4. 计算深度裁剪空间坐标
5. 将坐标映射到 NDC（归一化设备坐标）并采样深度纹理
6. 如果当前像素深度在已有深度之后（`depth_view_eye_z >= depth_hc.z`），保留颜色
7. 否则执行 `discard` 放弃该像素

这是后处理深度裁剪功能的 WGSL 实现，用于渲染到纹理后再按深度丢弃像素（如头发透明排序）。

#### 11. 顶点主函数 `vertex_main`

```wgsl
@vertex
fn vertex_main(in: VertexMainIn) -> VertexMainOut
```

1. 设置 `VIEW_ID`（来自 `in.view_index` 或硬编码为 0）
2. 从 `packed_geometry_N` 解包几何体字段到 `vb_xxx` 全局变量
3. 从 `packed_instance_N` 解包实例字段到 `dyninst_xxx`/`rustinst_xxx` 全局变量
4. 从 `_mp_dyn_uniforms` 拷贝动态 uniform 到 `uni_xxx` 全局变量
5. 从 `_mp_scope_uniforms` 拷贝作用域 uniform 到 `su_xxx` 全局变量
6. 初始化 `vtx_pos = vec4f(0,0,0,1)`
7. 调用 `io_vertex` 函数（若返回 `Vec4f` 则赋给 `vtx_pos`）
8. 展平所有 varying 字段的标量表达式，写入 `out_data.packed_varying_N`

#### 12. 片元主函数 `fragment_main`

分两个分支：无片元输出 vs 有片元输出。

**无片元输出**（`FragmentMainOut` 不存在）：
```wgsl
@fragment
fn fragment_main(in: FragmentMainIn)
```

1. 设置 `VIEW_ID`
2. 解包 varying 字段
3. 拷贝动态 uniform 和作用域 uniform
4. 调用 `io_fragment`

**有片元输出**（MRT 场景）：
```wgsl
@fragment
fn fragment_main(in: FragmentMainIn) -> FragmentMainOut
```

1. 同上步骤 1-4
2. 构建 `FragmentMainOut` 结构体
3. 根据输出类型生成默认初始化值（`vec4f` → `frag_fb0`，其他类型使用零值）
4. 返回结构体

片元输出类型到默认值的映射表：

输出类型 | 默认值
---------|------
`f32` | `f32(0.0)`
`i32` | `i32(0)`
`u32` | `u32(0)`
`vec2f` | `vec2f(0.0)`
`vec3f` | `vec3f(0.0)`
`vec4f` | `frag_fb0`
`vec2i` | `vec2i(0)`
`vec3i` | `vec3i(0)`
`vec4i` | `vec4i(0)`
`vec2u` | `vec2u(0u)`
`vec3u` | `vec3u(0u)`
`vec4u` | `vec4u(0u)`

### `compile_draw_shader_wgsl_source(vm, io_self, layout_source, xr_multiview) -> Result<WgslDrawShaderSource, String>`

WGSL 编译器的外部入口函数。流程：

1. 创建 `ShaderOutput` 实例，设置 `backend = ShaderBackend::Wgsl`
2. 调用 `pre_collect_rust_instance_io` 和 `pre_collect_shader_io` 执行 IO 预收集
3. 编译顶点函数 `vertex@`（若存在）
4. 编译片元函数 `fragment@`（若存在）
5. 检查编译错误
6. 从 `layout_source` 拷贝 IO 和采样器布局（与 Vulkan 主编译路径保持锁步，确保 `dyn/rust instance packing`、`uniform buffer bindings` 等与绘制映射一致）
7. 调用 `assign_uniform_buffer_indices(&heap, 3)` 分配 uniform buffer 绑定点（从 3 开始）
8. 计算几何体和实例的 slot 数用于返回
9. 调用 `build_draw_shader_wgsl` 生成完整 WGSL 源码
10. 返回 `WgslDrawShaderSource`，包含 wgsl 源码、绑定点元数据和 slot 数量

此函数的关键设计约束：WGSL 着色器的 IO 布局必须与主编译器路径（Vulkan SPIR-V 路径）的绘制映射完全一致，否则跨 API 的实例数据打包/解包会错位。因此 `output.io` 被来自 `layout_source` 的 IO 列表覆盖而非使用编译过程中自己收集的 IO。
