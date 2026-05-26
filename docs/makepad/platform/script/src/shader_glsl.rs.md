# `shader_glsl.rs` — OpenGL GLSL 后端代码生成器

## 概述

本文件实现了 Makepad 着色器编译器的 OpenGL/GLSL 后端，负责将中间表示（IR）翻译为 OpenGL 3.x/4.x 兼容的 GLSL 着色器源码。核心结构 `GlslPackedField` 管理顶点属性、实例属性和 varying 变量的打包布局。

**核心数据结构：**
- `GlslPackedFormat` — 枚举 `Float`/`UInt`/`SInt`，标识属性在 CPU→GPU 传输时的整型/浮点编码
- `GlslPackedSource` — 枚举 `NumericFloat`/`BitPackedFloat`，标识 varying 是常规浮点数还是位编码整数
- `GlslPackedField` — 记录每个字段的名称、Pod 类型、占用的 slot 数量和偏移量，用于打包/解包

所有方法都以 `impl ShaderOutput` 实现，此结构体是整个着色器编译器的核心输出上下文。

---

## 公有方法

### `glsl_create_vertex_shader(&self, vm: &ScriptVm, shared_defs: &str, out: &mut String)`

顶点着色器主入口。流程如下：

1. 调用 `glsl_collect_geometry_fields` 收集所有顶点缓冲（`VertexBuffer`）字段
2. 调用 `glsl_collect_instance_fields` 收集动态实例和 Rust 实例字段
3. 调用 `glsl_collect_varying_pack_fields` 收集需要传递给片元着色器的 varying 字段
4. 计算 varying 所需的总 slot 数（按 vec4 对齐）
5. 依次写入 `shared_defs`、uniform 块声明、纹理采样器声明、全局变量、顶点输入属性、varying 接口、`io_vertex` 函数和 `main` 函数体

### `glsl_create_fragment_shader(&self, vm: &ScriptVm, shared_defs: &str, out: &mut String)`

片元着色器主入口。与顶点版本对称：

1. 收集 varying 字段并计算 slot 数
2. 写入 `shared_defs`、uniform 块、纹理采样器、片元全局变量
3. 写入 varying 接口（使用 `in` 限定符）
4. 写入片元输出声明（`layout(location = N) out ...`）
5. 生成 `io_fragment` 函数和 `main` 函数体

---

## 私有方法

### `glsl_write_uniform_blocks(&self, vm, out)`

生成三种 GLSL uniform block 声明：

1. **UniformBuffer** — 对每个绑定为 `UniformBuffer` 的 IO 生成 `layout(std140) uniform` 块。`draw_pass` 有特殊处理，调用 `glsl_write_draw_pass_uniform_block`
2. **Uniform** — 所有常规 uniform 变量合并到 `uniform userUniforms` 块中
3. **ScopeUniform** — 所有作用域 uniform 合并到 `uniform liveUniforms` 块中

均使用 `std140` 布局以匹配 CPU 端内存布局。

### `glsl_write_draw_pass_uniform_block(&self, vm, ty, block_name, io_name, out) -> bool`

特殊处理 `draw_pass` uniform 块（即投影/视图矩阵）。核心逻辑：

1. 查找 Pod 类型中的 struct 字段
2. 对 `camera_projection`、`camera_view`、`depth_projection`、`depth_view`、`camera_inv` 这五个矩阵字段生成 `type name[2]` 数组声明（左右眼双份），对应的 `_r` 后缀字段被跳过（数组的元素之一）
3. 其他字段正常生成单变量声明

### `glsl_write_texture_uniforms(&self, out)`

遍历所有纹理 IO，为每个纹理输出对应的 `uniform samplerX tex_name;` 声明。纹理类型的 GLSL 映射由 `glsl_sampler_type` 方法处理。

### `glsl_write_vertex_globals(&self, vm, out)`

声明顶点着色器全局变量：

- `vtx_pos` — 顶点位置寄存器（`vec4`），供直接修改 `self.pos` 的着色器使用
- `vb_xxx` — 顶点缓冲的全局拷贝
- `dyninst_xxx` / `rustinst_xxx` — 实例数据的全局拷贝
- `var_xxx` — varying 的全局拷贝

片元着色器也有对称的 `glsl_write_fragment_globals`，额外包含 `frag_fbN` 片元输出变量。

### `glsl_write_vertex_input_attrs(&self, geometry_fields, instance_fields, out)`

将几何体和实例槽位按 `vec4` 分块，生成 `in vec4 packed_geometry_N` 和 `in vec4 packed_instance_N` 输入属性声明。使用 `glsl_num_packed_vec4s` 向上取整计算需要多少个 vec4。

### `glsl_write_varying_interface(&self, varying_slots, is_vertex, out)`

生成 varying 变量的接口声明。顶点 stage 使用 `out vec4 packed_varying_N`，片元 stage 使用 `in vec4 packed_varying_N`。varying 同样按 vec4 分块。

### `glsl_write_fragment_outputs(&self, vm, out)`

为每个 `FragmentOutput` 生成 `layout(location = index) out type _mp_frag_index;` 声明，用于 MRT（多渲染目标）。

### `glsl_write_vertex_main(&self, vm, geometry_fields, instance_fields, varying_fields, out)`

生成 `main()` 函数体：

1. 从 `packed_geometry_N` 解包所有几何体字段（调用 `glsl_unpack_field_to_statements`）
2. 从 `packed_instance_N` 解包所有实例字段
3. 初始化 `vtx_pos = vec4(0,0,0,1)`
4. 检查顶点函数 `vertex@` 的返回类型是否是 `Vec4f`，若是则直接取返回值赋给 `vtx_pos`，否则仅调用函数
5. 将所有 varying 字段展平为标量表达式，写入 `packed_varying_N` 的对应分量

### `glsl_write_fragment_main(&self, vm, varying_fields, out)`

生成片元 `main()` 函数体：

1. 从 `packed_varying_N` 解包所有 varying 字段
2. 调用 `io_fragment` 函数
3. 将 `frag_fbN` 变量拷贝到对应的片元输出 `_mp_frag_N`

### `glsl_write_functions_for_entries(&self, out, entries)`

通过可达性分析确定需要包含哪些函数定义：

1. 调用 `glsl_collect_reachable_functions` 从入口函数（如 `io_vertex`、`io_fragment`）出发做传递闭包
2. 只输出被标记为可达的函数的签名和函数体

这是代码裁剪的关键优化，避免 GLSL 编译器中未使用函数的告警。

### `glsl_collect_reachable_functions(&self, entries) -> BTreeSet<usize>`

基于工作列表（worklist）算法的函数可达性分析：

1. 先从函数签名中提取函数名（去掉参数列表）
2. 如果任何函数名提取失败，直接返回所有函数索引（保守策略）
3. 从入口函数名出发，逐层查找函数体内的调用，通过 `glsl_body_calls_function` 判断调用关系
4. 直到工作列表为空，返回最终可达的函数索引集

### `glsl_function_name_from_sig(call_sig) -> Option<String>`

解析函数签名字符串（如 `float my_func(vec3 a, float b)`），提取函数名部分 `my_func`。通过找到左括号位置，向前取最后一个 token。

### `glsl_body_calls_function(body, function_name) -> bool`

在函数体字符串中搜索 `functionName(` 模式来判断是否调用了该函数。边界检查：确保匹配位置前面的字符不是字母数字或下划线，避免匹配到子字符串中的假名（如 `myFunc` 中的 `Func`）。

### `glsl_collect_geometry_fields(&self, vm) -> Vec<GlslPackedField>`

收集所有 `VertexBuffer` 类型的 IO，调用 `glsl_push_field` 构建打包字段列表。偏移量从 0 开始顺序递增。

### `glsl_collect_instance_fields(&self, vm) -> Vec<GlslPackedField>`

收集所有 `DynInstance` 和 `RustInstance` 类型的 IO，顺序与声明一致。实例字段使用 attribute packing 模式（支持位编码整数）。

### `glsl_collect_varying_pack_fields(&self, vm) -> Vec<GlslPackedField>`

收集所有 `DynInstance`、`RustInstance` 和 `Varying` 类型的 IO，用于构建 varying 打包布局。此处使用 `attribute_packing = false`（非位编码模式，使用数值浮点数）。

### `glsl_push_field(&self, vm, io, prefix, attribute_packing, offset, out)`

核心打包逻辑：计算一个字段需要多少 slot（通过 `ScriptPodTy.slots()`），并根据打包格式在 multi-slot 整数类型前后进行 4 对齐。如果字段是 `UInt`/`SInt` 格式且占用超过 1 slot 且当前偏移不是 4 的倍数，则填充到 4 的倍数再开始，写入后也做同样对齐。

### `glsl_unpack_field_to_statements(&self, vm, field, prefix, source, out)`

为打包字段生成解包赋值语句。对于结构体类型的字段，为避免某些 GLES 驱动（如 Android 模拟器的 ANGLE/SwiftShader）拒绝 struct 构造函数语法，采用逐成员赋值方式：

- 结构体：展平每个子字段，逐行 `field.subfield = expression;`
- 非结构体：调用 `glsl_unpack_expr_for_field` 生成单个赋值表达式

### `glsl_reconstruct_from_scalars(&self, vm, ty, source, scalars, scalar_index) -> String`

类型化的标量重构入口，将 `ScriptPodType` 包装为 `ScriptPodTypeInline` 后委托给 `glsl_reconstruct_inline`。

### `glsl_reconstruct_inline(&self, vm, ty, source, scalars, scalar_index) -> String`

递归地从标量列表重构复合类型的构造函数表达式：

- **Struct**：递归构建每个字段的表达式，调用 `StructName(field1, field2, ...)`
- **Vec**：提取对应数量的标量，调用 `VecN(c1, c2, ...)`
- **Mat**：提取矩阵维度对应数量的标量，调用 `MatN(c1, c2, ...)`
- **Scalar**：直接从列表取值

### `glsl_take_scalar_or_zero(scalars, scalar_index) -> String`

从标量列表中取出当前位置的值，索引递增。如果超出列表范围，返回 `"0.0"` 作为默认值。

### `glsl_flatten_exprs(&self, vm, ty, expr, out)`

将一个复合类型的表达式展平为标量表达式列表的入口，包装后委托给 `glsl_flatten_inline`。

### `glsl_flatten_inline(&self, ty, expr, out)`

递归地将复合类型表达式拆解为标量分量：

- **Struct**：对每个字段生成 `(expr).field_name`
- **Vec**：对每个分量生成 `(expr).x/y/z/w`
- **Mat**：对每个元素生成 `(expr)[col][row]`
- **Scalar**：直接推送

非 f32 标量通过 `glsl_to_float_scalar_expr` 转换为浮点表达式。

### `glsl_to_float_scalar_expr(ty, expr) -> String`

将各种标量类型的表达式转为浮点表达式：

- `F32`/`F16`：原样返回
- `U32`/`I32`：包装为 `float(expr)`
- `Bool`：使用三元表达式 `((expr) ? 1.0 : 0.0)`

### `glsl_convert_scalar_expr(source, target, expr) -> String`

根据打包源和目标类型转换标量表达式：

- **NumericFloat 源**：f32 原样，u32→`uint(expr)`，i32→`int(expr)`，bool→`(expr != 0.0)`
- **BitPackedFloat 源**：f32 原样，u32→`floatBitsToUint(expr)`，i32→`floatBitsToInt(expr)`，bool→`(floatBitsToUint(expr) != 0u)`

位编码模式使用 `floatBitsToUint/floatBitsToInt` 来保持整数位模式的精确性，而不损失精度。

### `glsl_attr_format_from_pod_ty(ty) -> GlslPackedFormat`

根据 Pod 类型判断顶点属性的打包格式：

- 所有无符号和布尔标量/向量 → `UInt`
- 有符号标量/向量 → `SInt`
- 浮点类型和向量 → `Float`

### `glsl_uniform_block_name(&self, name) -> String`

将 IO 名称映射为 GLSL uniform block 名称：
- `draw_pass` → `passUniforms`
- `draw_list` → `draw_listUniforms`
- `draw_call` → `draw_callUniforms`
- 其他 → `{name}_Uniforms`

### `glsl_sampler_type(&self, tex_type) -> &'static str`

纹理类型到 GLSL 采样器类型的映射：

| TextureType | GLSL 类型 |
|------------|-----------|
| Texture1d | sampler2D |
| Texture1dArray | sampler2DArray |
| Texture2d | sampler2D |
| Texture2dArray | sampler2DArray |
| Texture3d | sampler3D |
| Texture3dArray | sampler3D |
| TextureCube | samplerCube |
| TextureCubeArray | samplerCubeArray |
| TextureDepth | sampler2D |
| TextureDepthArray | sampler2DArray |
| TextureVideo | samplerExternalOES（Android 非 Vulkan） / sampler2D |

`TextureVideo` 的 Android 特殊处理是因为 Android 平台使用 `GL_TEXTURE_EXTERNAL_OES` 扩展来处理视频帧，而标准 OpenGL ES 需要特殊的采样器类型。

### `glsl_num_packed_vec4s(slots) -> usize`

将 slot 数量转换为需要多少个 `vec4` 来存储：`(slots + 3) / 4`，即向上取整到 4 的倍数。

### `glsl_packed_component(prefix, slot) -> String`

将全局 slot 索引映射到具体的 `prefix{vec_idx}.{comp}` 表达式。vec_idx = slot / 4，分量索引 = slot % 4。

### `glsl_swizzle_component(index) -> &'static str`

数字索引 0/1/2/3 映射到 GLSL swizzle 字符 x/y/z/w。

### `glsl_type_name_from_ty(&self, vm, ty) -> String`

通过 VM 的 heap 查询 Pod 类型的名称，委托给 `back-end.pod_type_name_from_ty`。

### `glsl_type_name_inline(&self, ty) -> String`

获取内联类型的 GLSL 名称。对于命名的结构体类型，使用 `map_pod_name` 映射；对于匿名结构体，生成 `S{index}` 格式的占位名；其他类型委托给 `back-end.pod_type_name`。
