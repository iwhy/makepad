# shader_backend.rs — 着色器后端代码生成与类型映射

**文件路径**: `platform/script/src/shader_backend.rs`  
**行数**: 1668 行  
**作用**: 定义 Makepad 着色器编译器的后端抽象层。负责将中间表示（IR）翻译为具体 GPU 着色器语言（Metal、WGSL、HLSL、GLSL）以及 Rust CPU 回退后端的代码生成。核心职责包括：IO 声明前缀、类型名称映射、变量声明语法、零值字面量、内建函数名映射、Pod 结构体定义生成。

---

## 主要类型定义

### `ShaderBackend`（第 14–21 行）
着色器后端枚举，定义了五个目标：
- **`Metal`**: Apple Metal Shading Language
- **`Wgsl`**: WebGPU Shading Language
- **`Hlsl`**: DirectX HLSL
- **`Glsl`**: OpenGL GLSL
- **`Rust`**: Rust CPU 回退运行时（用于无 GPU 环境）

默认后端为 Metal。

### `ShaderIoPrefix`（第 23–28 行）
IO 前缀的三种表示方式：
- **`Prefix(&'static str)`**: 静态字符串前缀（如 `"_io."`）
- **`Full(&'static str)`**: 完整静态字符串
- **`FullOwned(String)`**: 运行时计算的完整字符串（如带有动态索引的片段输出）

---

## `ShaderBackend` 方法实现

### `get_shader_io_kind_and_prefix`（第 31–580 行）
将 `ShaderIoType`（如 `SHADER_IO_TEXTURE_2D`）映射为 `(ShaderIoKind, ShaderIoPrefix)` 元组，决定生成的着色器代码中如何引用 IO 资源。按后端和着色器阶段分类：

**Metal 顶点阶段**:
- Rust 实例/动态实例 → `_io.i->` 前缀
- 动态统一变量 → `_io.u->`
- 统一缓冲区 → `_io.u_`
- 插值器变量（Varying） → `_iov.v->`
- 顶点缓冲区 → `_io.vb[_iov.vid].`
- 纹理/采样器 → `_io.` 前缀
- 作用域统一变量 → `_io.su->`
- 顶点位置 → 完整 `Full` 表达式 `_iov.v->_position`
- 片段输出 → 空前缀（仅在顶点阶段返回）

**Metal 片段阶段**:
- 片段输出（`SHADER_IO_FRAGMENT_OUTPUT_0 ~ MAX`）→ `_iof.fb->fbN` 格式
- Rust 实例 → `_io.i->`
- 插值器 → `_iof.v->`
- 其余与顶点阶段类似，纹理仍为 `_io.` 前缀

**HLSL 阶段**:
- 实例数据 → `_mp_iov.i.` 前缀
- 统一变量/缓冲区 → `u_` 前缀
- 插值器 → `_mp_iov.v.` 前缀
- 纹理 → 空前缀（HLSL 使用全局绑定）
- 作用域统一变量 → `su_` 前缀
- 顶点缓冲区 → `_mp_iov.vb.`
- 片段输出 → `_iofb.fbN` / `_mp_iof.fbN`

**Rust 阶段**:
- 所有 IO 使用 `rcx.` 前缀（`RenderCx` 上下文引用）
- Rust 实例 → `rcx.rustinst_`
- 动态实例 → `rcx.dyninst_`
- 统一变量 → `rcx.uni_`
- 统一缓冲区 → `rcx.unibuf_`
- 插值器 → `rcx.var_`
- 纹理 → `rcx.tex_`
- 采样器 → `rcx.sampler_`
- 作用域统一变量 → `rcx.su_`
- 片段输出 → `rcx.frag_fbN`
- 顶点位置 → `rcx.vtx_pos`

**GLSL/WGSL 阶段**:
- 统一前缀风格：`rustinst_`、`dyninst_`、`uni_`、`unibuf_`、`var_`、`tex_`、`sampler_`、`su_`、`vb_`
- 顶点位置 → `vtx_pos`
- 片段输出 → `frag_fbN`

### `get_io_all`（第 582–589 行）
返回 IO 集合的访问表达式。Metal → `_io`，Rust → `rcx`，HLSL/GLSL/WGSL → 空字符串。

### `get_io_all_decl`（第 591–598 行）
返回 IO 集合的参数声明。Metal → `thread Io &_io`，Rust → `rcx: &mut RenderCx`，HLSL → 无。

### `get_io_self`（第 600–615 行）
返回当前着色器阶段的自引用表达式。Metal → `_iov`（顶点）/ `_iof`（片段），其余后端返回空。

### `get_io_self_decl`（第 617–632 行）
返回自引用的参数声明。Metal 返回 `thread IoV &_iov`（顶点）/ `thread IoF &_iof`（片段）。

### `map_local_name`（第 634–706 行）
将 LiveId 局部变量名映射为后端着色器语言的有效标识符：
- **HLSL**: `l_{id}_{shadow}` 格式
- **GLSL**: `l_{base}_{shadow}`，`self` → `_self`
- **Rust**: 直接使用原始名，但检测 Rust 关键字（`type`、`match`、`fn`、`let`、`mut`、`return`、`struct` 等 26 个）并加 `r#` 前缀。`self` → `_self`
- **Metal/WGSL**: `_s{shadow}{id}`，`self` → `_self`

影子变量通过 `{shadow}` 后缀区分。

### `map_param_name`（第 708–727 行）
映射函数参数名。`self` 参数特殊处理：Rust 和 WGSL 中 `self` 是指针，需解引用为 `(*_self)`。HLSL/GLSL 使用 `p_{id}_{shadow}` 格式，其他后端复用 `map_local_name`。

### `map_function_name`（第 729–735 行）
映射函数名。HLSL 添加 `f_` 前缀以避免与关键字冲突。Rust 和其他后端直接保留原名。

### `map_io_name`（第 737–743 行）
映射 IO 资源名。HLSL 添加 `io_` 前缀。

### `map_field_name`（第 745–747 行）
字段名映射，委托给 `map_field_name_typed(id, true)`。

### `map_field_name_typed`（第 751–804 行）
带类型感知的字段名映射，处理向量类型的 swizzle 操作：
- **HLSL**: 检测 swizzle 字符（xyzw/rgba 组合）直接保留，非 swizzle 字段加 `f_` 前缀
- **Rust**: 检测 swizzle 模式。多分量 swizzle（如 `.xy`）映射为方法调用 `.xy()`；单分量（`.r` → `.x`）映射为 xyzw 命名。rgba 到 xyzw 的转换通过字符级映射实现
- **其他后端**: 直接输出 LiveId 字符串

### `write_var_decl`（第 809–822 行）
生成变量声明语句。各后端语法：
- **Metal/HLSL/GLSL**: `type_name var_name;\n`
- **WGSL**: `var var_name:type_name;\n`
- **Rust**: `let mut var_name: type_name = zero_literal;\n`（需零初始化）

### `write_var_decl_zero_init`（第 827–847 行）
生成带零初始化的变量声明。与 `write_var_decl` 的区别在于 HW 后端也生成零值（如 `type_name(0)`），而 Rust 总是需要初始化。

### `zero_literal`（第 850–1011 行）
为给定类型名生成零值字面量。按后端分别实现：

**Metal**:
- 标量: `0.0`（float）、`0.0h`（half）、`0`（uint/int）、`false`（bool）
- 向量: `float2(0)` 等构造函数
- 矩阵: `float4x4(0.0)` 等
- 默认: `T()` 空构造函数

**HLSL**:
- 标量与 Metal 类似
- 向量: `float2(0.0, 0.0)` 各个分量展开
- 矩阵: `float4x4(0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0)` 16 个分量
- 内部使用 `join_n` 辅助函数生成重复分量

**GLSL**:
- 标量: `0.0`、`0u`、`0`、`false`
- 向量: `vec4(0.0)` 等单参数构造函数
- 矩阵: `mat4(0.0)` 等

**WGSL**:
- 标量: `0.0`、`0.0h`、`0u`、`0i`、`false`
- 向量: `vec4f()` 空构造函数（WGSL 允许空构造表示零）
- 矩阵: `mat4x4f()` 空构造函数

**Rust**:
- 标量: `0.0f32`、`0u32`、`0i32`、`false`
- 向量: `vec2(0.0, 0.0)` 等函数调用
- 矩阵: `Mat4f::default()`
- 默认: `T::default()`

### `register_ids`（第 1013–1123 行）
为每个后端注册 LiveId 查找表中的类型名和内建函数名。确保后端特有的标识符在 `id!` 宏中可解析。

**Metal/HLSL**: 注册 `float`、`half`、`uint`、`int` 标量，`float2`~`float4` 等向量，`float2x2`~`float4x4` 矩阵，`atomic_uint/int`，`packed_*` 紧凑类型，以及内建函数名（`dfdx`、`dfdy`、`ddx`、`ddy`、`_mp_inverse`、`rsqrt`、`fmod`、`frac`、`lerp`、`discard_fragment`）。

**GLSL**: 注册 `float/uint/int`、`vec2~vec4/ivec2~ivec4/uvec2~uvec4/bvec2~bvec4`、`mat2~mat4`，内建 `dFdx/dFdy/inverse/inversesqrt/mod`。

**WGSL**: 注册 `dpdx/dpdy/inverse`。

**Rust**: 注册 `inverse`、`inverseSqrt`、`modf`。

### `map_builtin_name`（第 1125–1175 行）
将内建函数的规范名映射为后端特有名称：

**Metal**:
- `dFdx` → `dfdx`、`dFdy` → `dfdy`
- `inverse` → `_mp_inverse`（`inverse` 在 Metal 中不是标准函数）
- `inverseSqrt` → `rsqrt`
- `modf` → `fmod`
- `discard` → `discard_fragment`

**HLSL**:
- `dFdx` → `ddx`、`dFdy` → `ddy`
- `inverseSqrt` → `rsqrt`
- `modf` → `fmod`
- `fract` → `frac`
- `mix` → `lerp`

**GLSL**:
- `inverseSqrt` → `inversesqrt`
- `modf` → `mod`（内置取模函数）
- `atan2` → `atan`（GLSL 的 `atan(y, x)` 实现 atan2）

**WGSL**:
- `dFdx` → `dpdx`、`dFdy` → `dpdy`

**Rust**:
- 所有名称保留不变，`dFdx/dFdy/discard` 在 CPU 运行时是无操作。

### `map_packed_pod_name`（第 1179–1218 行）
将 Pod 类型名映射为 Metal 紧凑类型（`packed_*`），确保 CPU 端 `repr(C)` 结构体与 GPU 实例缓冲区的内存布局对齐。非 Metal 后端委托给 `map_pod_name`。

### `map_pod_name`（第 1220–1285 行）
将 Makepad 内部的规范 Pod 类型名（如 `f32`、`vec2f`、`mat4x4f`）映射为具体后端类型名：

**Metal/HLSL**:
- `f32` → `float`、`f16` → `half`、`u32` → `uint`、`i32` → `int`
- `vec2f/3f/4f` → `float2/3/4`，`vec2h/3h/4h` → `half2/3/4`
- `vec2u/3u/4u` → `uint2/3/4`，`vec2i/3i/4i` → `int2/3/4`
- `vec2b/3b/4b` → `bool2/3/4`
- `mat2x2f~mat4x4f` → `float2x2~float4x4`
- `atomic_u32/i32` → `atomic_uint/int`

**GLSL**:
- `vec2f` → `vec2`，`vec2h` → `vec2`（half 映射为 float）
- `mat2x2f` → `mat2`，`mat3x3f` → `mat3`，`mat4x4f` → `mat4`

**WGSL/Rust**: 保留原名（`vec2f`、`mat4x4f` 等）。

### `pod_struct_defs`（第 1287–1307 行）
生成所有根的 Pod 结构体定义（plain 版本）。先通过拓扑排序确定依赖顺序（`pod_struct_visit`），再逐一定义。

### `pod_struct_defs_mixed`（第 1309–1343 行）
同时生成 packed（压缩）和 plain（普通）两种版本的结构体定义。Packed 版本优先（用于 GPU 实例缓冲区），plain 版本随后（用于其他 IO）。通过 `BTreeSet` 避免重复访问已处理的结构体类型。

### `pod_struct_visit`（第 1345–1379 行）
递归遍历结构体类型的依赖图，使用 `visited` 集合防止环，生成拓扑排序的 `order` 列表。收集结构体字段中引用的子类型。

### `pod_type_def`（第 1381–1389 行）
为单个 Pod 类型生成结构体定义。委托给 `pod_type_def_impl`。

### `pod_type_def_impl`（第 1391–1483 行）
结构体定义核心实现。按后端生成不同语法：

- **Rust**: 添加 `#[derive(Default, Clone, Copy)]` 和 `#[repr(C)]`，使用 `pub field: TypeName` 格式
- **Metal/HLSL/GLSL**: 使用 `TypeName field_name;` 格式，支持固定数组的 `TypeName field_name[N];` 语法
- **WGSL**: 使用 `field_name: TypeName,` 格式，末尾逗号

另外，HLSL 后端会额外生成构造函数（`consfn_*`）以支持方便的结构体初始化。

对字段类型为 FixedArray 时调用 `pod_type_def_metal_array` 处理数组维度的递归展开。

### `pod_type_def_metal_array`（第 1485–1511 行）
处理 Metal 风格的多维固定数组定义。递归展开所有 FixedArray 层，生成 `TypeName field_name[size1][size2]...;` 格式。

### `pod_type_name_referenced`（第 1513–1538 行）
输出类型名称并将引用的结构体类型加入 `referenced` 集合。处理三种类型：
- `Struct` → 输出名称并加入集合
- `FixedArray` → 输出 `array<InnerType, N>`（WGSL 风格）
- `VariableArray` → 输出 `array<InnerType>`
- 其他类型 → 委托给 `pod_type_name`

### `pod_type_name_packed_referenced`（第 1540–1565 行）
`pod_type_name_referenced` 的 packed 版本。使用 `pod_type_name_packed` 输出类型名。

### `pod_type_name_from_ty`（第 1567–1574 行）
从 `ScriptPodType` 索引构造 `ScriptPodTypeInline` 临时实例后输出类型名。

### `pod_type_name_packed_from_ty`（第 1578–1590 行）
packed 版本的 `pod_type_name_from_ty`，用于 Metal 实例缓冲区结构体。

### `pod_type_name_packed`（第 1593–1619 行）
输出 packed 类型名称。仅对基本数值类型（F32、F16、U32、I32、Bool）和 Vec/Mat 类型使用 `map_packed_pod_name` 映射，其他类型回退到 `pod_type_name`。

### `pod_type_name`（第 1621–1667 行）
通用类型名称输出。按类型分类：
- **F32/F16/U32/I32/Bool**: 使用 `map_pod_name` 映射
- **AtomicU32/AtomicI32**: 输出 `atomic<uint>` / `atomic<int>`（WGSL 风格）
- **Vec/Mat**: 使用 `map_pod_name(v.name())` 映射
- **Struct**: 使用自定义名称
- **FixedArray**: `array<InnerType, N>`
- **VariableArray**: `array<InnerType>`
- **默认**: 输出 `"unknown"`

---

## 文件职责总结

`shader_backend.rs` 是 Makepad 着色器编译器的后端代码生成抽象层，核心职责包括：

1. **后端枚举**: 定义 Metal、WGSL、HLSL、GLSL、Rust 五个目标后端的类型系统
2. **IO 前缀管理**: 根据后端和着色器阶段（顶点/片段）生成正确的 IO 资源访问前缀
3. **名称映射**: 类型名、变量名、函数名、字段名、内建函数名的跨后端映射
4. **声明语法**: 变量声明、零初始化、结构体定义的跨后端代码生成
5. **内建函数映射**: 同一内置函数在不同后端的名称差异处理（如 mix↔lerp、fract↔frac）
6. **结构体定义生成**: 递归遍历 Pod 类型依赖图，生成带拓扑排序的结构体定义代码
7. **紧凑类型支持**: Metal packed 类型用于 CPU-GPU 实例缓冲区的内存对齐
