# shader_calls.rs — 函数/方法调用编译

**源码路径:** `platform/script/src/shader_calls.rs`  
**总行数:** 2362 行  
**核心职责:** 将 Splash 字节码中的函数调用、方法调用、Pod/数组构造函数调用、内置函数调用、纹理方法调用翻译为目标后端着色器代码（WGSL/Metal/HLSL/GLSL/Rust）。

---

## 核心类型与架构

所有方法均在 `ShaderFnCompiler` 的 impl 块中。编译流程的核心思路是：

1. `CALL_ARGS` / `CALL_METHOD_ARGS` 字节码首先解析调用目标和参数，向 `self.mes` 栈（ME = Meta Expression stack）压入一个调用类型标记（如 `ShaderMe::ScriptCall`、`ShaderMe::BuiltinCall`、`ShaderMe::Pod{}` 等）。
2. 后续的 `CALL_EXEC` 字节码从 `self.mes` 栈弹出标记，执行实际代码生成，将结果字符串推回 `self.stack`。
3. 这种两阶段设计（setup + exec）是因为参数需要在调用前逐个压栈，编译器需要先收集参数再统一生成代码。

---

## 方法详解

### `handle_pod_type_call` — Pod类型构造调用（第21-47行）

从栈上弹出参数规模信息后，检查目标是否为 `ArrayBuilder` 类型。如果是数组构造器，压入 `ShaderMe::ArrayConstruct` 标记并提前返回。否则确保类型名称已记录（调用 `ensure_struct_name`），然后压入 `ShaderMe::Pod` 标记，其中包含 `pod_ty` 和空 args 向量，后续由 `CALL_EXEC` 填充并执行。

### `handle_call_args` — 函数调用参数解析（第49-108行）

这是最核心的调度枢纽。从栈顶弹出目标类型，按优先级依次检查：
1. **shader_scope 中的 PodType** → 委托 `handle_pod_type_call` 处理构造调用
2. **脚本 scope 中的 PodType** → 同上
3. **脚本 scope 中的函数对象**：
   - `ScriptFnPtr::Script`（自定义 Splash 函数）→ 压入 `ShaderMe::ScriptCall`，记录函数对象 `fnobj`、名称、当前 IO 前缀字符串和 `sself` 类型
   - `ScriptFnPtr::Native`（内置函数）→ 压入 `ShaderMe::BuiltinCall`，记录函数指针和名称
4. 以上均不匹配 → 报告"shader call target is not a function"错误

### `handle_array_construct` — 数组构造函数代码生成（第110-206行）

根据元素类型和数量构造固定长度数组。对每个后端生成不同的语法：
- **WGSL**: `array<T, N>(e1, e2, ...)`
- **Metal/HLSL**: `{e1, e2, ...}`（花括号初始化）
- **GLSL/Rust**: `T[N](e1, e2, ...)`

### `handle_pod_construct` — Pod类型（向量/矩阵/结构体）构造函数（第208-501行）

这是最复杂的构造逻辑，支持命名参数和位置参数两种模式：

**命名参数路径**（第277-357行）：当第一个参数带有 `.name` 时，按结构体字段顺序遍历各字段，对每个字段在 args 中按名查找对应的参数值。对每个参数执行类型匹配检查（Pod类型、Id变量、抽象整数/浮点数与期望字段类型的兼容性）。Rust 后端对结构体使用 `Struct { field: value }` 语法，其他后端使用逗号分隔。

**位置参数路径**（第358-467行）：按参数位置顺序处理。对参数执行 `pod_check_constructor_arg` 检查偏移量对齐和类型兼容。特殊处理 HLSL 后端单参数构造时的 splat 展开（如 `float4(1.0)` → `float4(1.0, 1.0, 1.0, 1.0)`）。对 Rust 后端结构体类型使用 `Struct { field0: v0, field1: v1 }` 命名语法。

**后端语法差异**（第469-493行）：
- WGSL/GLSL/HLSL: `Type(e1, e2, ...)`
- Metal 结构体: `{e1, e2, ...}`，非结构体: `Type(e1, e2, ...)`
- Rust 结构体: `Type { f0: v0, f1: v1 }`，非结构体: `Type(e1, e2, ...)`

最终从 `self.stack.new_string()` 分配输出缓冲区，将结果字符串和 Pod 类型推回栈。

### `compile_shader_def` — 着色器函数定义编译（第503-816行）

静态方法，主流程：
1. **构造函数名前缀** `method_name_prefix`：根据 `sself` 类型生成前缀（Pod 类型名 + `_`、`io_`、`scope{N}_`）
2. **第一遍参数类型解析**（第530-565行）：遍历函数对象 `fnobj` 的 vec 键值对，跳过 `self` 参数。对每个参数将 `AbstractInt`/`AbstractFloat` 决议为声明的具体 Pod 类型（如传入了抽象整数但函数声明期望 `f32`，则决议为 `pod_f32`）。收集为 `resolved_args`。
3. **参数个数校验**（第568-579行）：调用方提供的参数数量必须与函数声明匹配。
4. **重载检测与复用**（第582-601行）：若已有相同 `fnobj` 和参数类型的已编译函数，直接复用其名称签名，避免重复编译。
5. **生成参数列表** `fn_args`（第618-743行）：
   - 添加 IO 全局参数（`get_io_all_decl`）
   - 根据 `sself` 类型添加 self 参数（每种后端使用不同语法：WGSL `ptr<function, T>`、Metal `thread T&`、HLSL `inout T`、GLSL `inout T`、Rust `*mut T`）
   - 对每个函数参数生成类型标注声明
6. **编译函数体**（第749-815行）：递归检查防止无限递归（`recur_block`），调用 `compiler.compile_fn` 编译函数体。根据后端语法生成函数声明签名 `call_sig`（WGSL `fn name(args) -> Ret`，其他后端 `Ret name(args)`）。将 `ShaderFn` 推进 `output.functions`，返回返回类型和函数调用字符串（末尾已加左括号）。

### `handle_script_call` — 执行脚本函数调用（第818-841行）

在 `compile_shader_def` 已生成函数定义的基础上，完成调用表达式的拼接。对 GLSL/Rust 后端额外调用 `glsl_rewrite_call_args` 处理整数→浮点数的字面量重写（如 `2` → `2.0`）。最终将 `fn_name(args)` 字符串和返回类型推到栈上。

### `resolve_script_call_arg_types` — 解析函数调用参数类型（第843-878行）

静态辅助方法，遍历函数对象的参数声明，对 `AbstractInt`/`AbstractFloat` 类型的调用参数按其声明类型进行具体化决议。与 `compile_shader_def` 的第一遍参数解析逻辑相同但独立存在，用于调用方的参数类型准备。

### `glsl_rewrite_call_args` — GLSL/Rust 调用参数重写（第880-912行）

GLSL ES 不允许隐式 int→float 提升。此方法在调用参数中检测 `AbstractInt` 类型的参数，如果其决议类型是浮点数但传递的是简单整数字面量（如 `2`），重写为 `2.0`。使用 `split_call_args_top_level` 分割顶层逗号分隔的参数，保持括号/花括号/方括号嵌套不变。

### `split_call_args_top_level` — 顶层参数分割（第914-944行）

解析逗号分隔的参数列表，但只在 `paren_depth == 0 && bracket_depth == 0 && brace_depth == 0` 时拆分，正确处理嵌套结构。返回 `Vec<String>`。

### `is_simple_int_literal` — 判断简单整数字面量（第946-951行）

检查字符串是否只包含 ASCII 数字、`-`、`+` 字符。

### `handle_call_exec` — CALL_EXEC 字节码处理器（第953-1029行）

调用的执行阶段入口。首先检查 `self.mes` 栈顶是否为调用相关的 ME 类型，如果不是则说明调用 setup 阶段失败（如 `CALL_ARGS` 未成功压入 ME），此时推一个哑值到 stack 上保持栈平衡。然后根据 ME 类型分派：
- `ArrayConstruct` → `handle_array_construct`
- `Pod` → `handle_pod_construct`
- `ScriptCall` → `handle_script_call`
- `TextureBuiltin` → `handle_texture_builtin_exec`
- `BuiltinCall` / `PodBuiltinMethod` → `handle_builtin_call`

### `handle_builtin_call` — 内置函数调用的后端代码生成（第1032-1428行）

这是 Largest 的方法之一，处理所有内置着色器函数：

**特殊函数处理：**
- **`discard()`**（第1042-1058行）：Metal 输出 `discard_fragment()`，GLSL/WGSL/HLSL 输出 `discard`，Rust 输出 `{ rcx.discard = 1.0; return }`
- **`depth_clip(near, val, far)`**（第1061-1098行）：GLSL/WGSL 调用辅助函数 `depth_clip(...)`，Metal/HLSL/Rust 直接输出中间值
- **`asuint` / `asint` / `asfloat`**（第1101-1212行）：类型重新解释转换，每种后端使用不同 API：
  - GLSL: `floatBitsToUint` / `floatBitsToInt` / `intBitsToFloat` / `uintBitsToFloat`
  - WGSL: `bitcast<u32>` / `bitcast<i32>` / `bitcast<f32>`
  - HLSL: `asuint` / `asint` / `asfloat`
  - Metal: `as_type<uint>` / `as_type<int>` / `as_type<float>`
  - Rust: `.to_bits()` / `f32::from_bits`

**通用内置函数处理**（第1214-1427行）：
1. 检测参数中是否有浮点类型（`has_float`），如果有则将 `AbstractInt` 参数格式化为浮点数（追加 `.0`）
2. 通过 `output.backend.map_builtin_name(name)` 获取后端映射后的函数名
3. HLSL 特殊：单参数构造时的 splat 展开处理（如 `float4(1.0)` → 四个参数）
4. 标记 `output.uses_derivatives`：检测 `dFdx`/`dFdy` 调用
5. WGSL 特殊：`modf(x, y)`（浮点取模）在 WGSL 中没有对应的 `modf`（WGSL 的 `modf` 是分解小数部分），直接展开为 `(x) % (y)`
6. **Rust 后端的 dFdx/dFdy 实现**（第1298-1373行）：这是最复杂的部分。Rust 头端着色器是 CPU 模拟的，通过三通道方法计算屏幕空间导数：
   - 通道 0（dx pass）：将值存入 `quad_dx_buf`
   - 通道 1（dy pass）：将值存入 `quad_dy_buf`
   - 通道 2（compute pass）：计算当前值减存储值的差
   - 根据 `quad_lane_x`/`quad_lane_y` 判断方向，处理 1/2/3/4 通道（scalar/vec2/vec3/vec4）情况
7. **Rust 内联类型后缀**（第1376-1413行）：对 `clamp`、`max`、`min`、`abs`、`length`、`dot`、`normalize` 等函数，Rust 后端根据第一个参数类型追加类型后缀（如 `clamp_2f`、`max_4f`），用于 Rust 的静态派发

### `handle_texture_builtin_exec` — 纹理内置方法代码生成（第1430-1712行）

处理纹理类型的方法调用（`.size()`、`.sample()`、`.sample_as_bgra()`、`.sample_lod()`、`.sample_nearest()`、`.sample_video()`）：

**`size()`**（第1441-1481行）：
- Metal: `float2(texture.get_width(), texture.get_height())`
- WGSL: `vec2f(textureDimensions(texture))`
- HLSL: `_mpTexSize2D(texture)`（辅助函数，设置 `hlsl_needs_tex_size = true`）
- GLSL: `vec2(textureSize(texture, 0))`
- Rust: `vec2(texture.width as f32, texture.height as f32)`

**`sample()` / `sample_as_bgra()` / `sample_lod()` / `sample_nearest()`**（第1483-1625行）：
- 验证参数个数（`sample_lod` 需要 2 个参数，其余需要 1 个）
- 创建或获取采样器（`sample_nearest` 使用 Nearest 过滤，其余使用 Linear 默认值）
- Metal: `texture.sample(_s{N}, coord, level(lod))`
- WGSL: `textureSample(texture, _s{N}, coord)` / `textureSampleLevel(texture, _s{N}, coord, lod)`
- HLSL: `texture.SampleLevel(_s{N}, coord, lod)`
- GLSL: 通过辅助函数 `sample2d`/`sample2d_bgra`/`samplecube`/`samplecube_bgra`/`sample2d_lod`/`samplecube_lod`，并通过 `bind_texture_sampler` 记录纹理与采样器的绑定关系
- Rust: `texture.sample(coord)` / `texture.sample_lod(coord, lod)`，`sample_as_bgra` 是 `sample` 的别名

**`sample_video()`**（第1626-1689行）：视频纹理采样。Android GLSL 使用 `sample2dOES`，其他 GLSL 使用 `sample2d`。Metal 使用视频采样器，HLSL 使用 `SampleLevel`，WGSL 使用 `textureSampleLevel`，Rust 直接 `.sample()`。

### `handle_method_call_args` — 方法调用参数解析（第1714-1888行）

分派方法调用的中央路由。从栈上弹出方法名和 self 对象后，根据 self 的类型走不同路径：

1. **Texture 类型**（第1726-1731行）：调用 `handle_texture_method_call_args` 直接创建 `TextureBuiltin` ME
2. **ScopeTexture 类型**（第1734-1739行）：同上
3. **Id 类型**（第1741-1838行）：在 shader_scope 中查找变量：
   - 解析为 `IoSelf` → `handle_io_self_method_call_args`
   - 解析为 Pod 类型 → `handle_pod_method_call_args`
   - 解析为 PodType（静态方法）→ `handle_pod_type_method_call_args`
   - 解析为 ScopeObject → `handle_scope_object_method_call_by_id`
   - 解析为 ScopeTexture → `handle_scope_texture_method_call_by_id`
4. **直接的 Pod 类型**（第1842-1869行）：直接调用 `handle_pod_method_call_args`

### `handle_io_self_method_call_args` — IoSelf 方法调用参数（第1890-1930行）

在 io_self 对象的 prototype 链上查找方法。找到方法后：
- 构造调用前缀字符串（`get_io_all` + `get_io_self`）
- 压入 `ShaderMe::ScriptCall` 标记，sself 设为 `ShaderType::IoSelf(obj)`
- 调用 `maybe_pop_to_me` 进入参数收集阶段

### `handle_scope_object_method_call_args` — ScopeObject 方法调用参数（第1932-1973行）

在 scope object 上查找方法。ScopeObject 的方法没有 `_self` 参数——对 `self` 的引用在编译时解析为 `IoScopeUniform` 访问。只传递 `io_all` 参数。

### `handle_scope_object_method_call_by_id` — 通过名称查找 ScopeObject 方法（第1978-2008行）

在脚本 scope 中通过名称查找对象，确保不是 `shader_io` 类型也不是函数，然后委托给 `handle_scope_object_method_call_args`。

### `handle_scope_texture_method_call_by_id` — 通过名称查找 ScopeTexture 方法（第2013-2116行）

在脚本 scope 中查找纹理变量，确认是纹理 `shader_io` 类型。如果该 scope texture 尚未注册到 `output.scope_textures` 和 `output.io`，则生成唯一名称并注册。根据后端映射获取纹理访问前缀，然后创建 `ShaderMe::TextureBuiltin` 标记等待参数收集。

### `handle_pod_method_call_args` — Pod 类型实例方法调用（第2118-2215行）

先检查是否是 `is_pod_builtin_method`（`mix`、`clamp`、`smoothstep`、`step`、`min`、`max`），这些会翻译为内置函数调用（如 `x.mix(y, a)` → `mix(x, y, a)`），压入 `ShaderMe::PodBuiltinMethod` 标记。

否则在 Pod 类型的 prototype 对象上查找方法：
- `ScriptFnPtr::Script` → 构造 self 参数传递方式（每种后端不同：WGSL `&_self`，Metal 直接传引用，HLSL/GLSL 直接传值，Rust 传 `*mut` 指针），压入 `ShaderMe::ScriptCall`
- `ScriptFnPtr::Native` → 将 self 作为第一个参数，压入 `ShaderMe::BuiltinCall`

### `is_pod_builtin_method` — 判断是否为 Pod 内置方法名（第2218-2225行）

匹配 `mix`、`clamp`、`smoothstep`、`step`、`min`、`max`。

### `handle_pod_type_method_call_args` — PodType 静态方法调用（第2227-2278行）

在脚本 scope 中查找名称对应的 PodType，在其 prototype 对象上查找静态方法。找到后根据函数类型（Script/Native）压入对应的 ME 标记。

### `handle_texture_method_call_args` — 纹理方法调用参数准备（第2280-2296行）

简单的转发方法，将纹理表达式和方法 ID 封装为 `ShaderMe::TextureBuiltin` 标记压入 ME 栈。

### `rust_expand_pod_construct` — Rust 后端异构构造展开（第2301-2361行）

将异构 Pod 构造展开为平面标量列表。例如 `vec4(vec3(x,y,z), f32)` → `vec4(x, y, z, f32)`。对每个参数按 slot 数展开（scalar=1 直接输出、vec2 展开 xy、vec3 展开 xyz、vec4 展开 xyzw）。如果展开后组件数不足则重复最后一个值补全（splat）。
