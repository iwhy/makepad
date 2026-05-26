# draw_shader.rs — 着色器实例化、选项与映射管理

## 概述

`draw_shader.rs` 管理 Makepad 绘制管线中的着色器端到端生命周期。它定义了着色器选项 (`CxDrawShaderOptions`)、着色器存储 (`CxDrawShaders`)、着色器映射 (`CxDrawShaderMapping`) 以及着色器输入布局 (`DrawShaderInputs`)，负责将脚本层的着色器定义编译为平台无关的 GPU 输入布局。

---

## `CxDrawShaderOptions`（第 25–74 行）

绘制调用的着色器选项，控制 GPU 管线的固定功能状态：
- `draw_call_group: LiveId` — 绘制调用分组 ID，用于合并优化
- `debug_id: Option<LiveId>` — 调试标识
- `depth_write: bool` — 是否写入深度缓冲区（默认 true）
- `alpha_blend: bool` — 是否启用 Alpha 混合（默认 true）
- `backface_culling: bool` — 是否启用背面剔除（默认 false）

**`_appendable_drawcall()`** — 判断两个选项集合是否完全相同，决定两个绘制调用能否合并。目前直接比较 `self == other`，即所有字段必须相等。

---

## `CxDrawShaders`（第 83–104 行）

着色器的全局存储和缓存：
- `shaders: Vec<CxDrawShader>` — 所有已注册的着色器
- `os_shaders: Vec<CxOsDrawShader>` — 平台相关的着色器后端数据
- `compile_set: BTreeSet<usize>` — 需要编译的着色器索引集合
- `cache_object_reuse_epoch_seen: u64` — 缓存淘汰的代次标记
- `cache_object_id_to_shader: HashMap<ScriptObject, DrawShaderId>` — 从脚本对象到着色器 ID 的缓存
- `cache_functions_to_shader: LiveIdMap<LiveId, DrawShaderId>` — 从函数哈希到着色器 ID 的缓存
- `cache_code_to_shader: HashMap<CxDrawShaderCode, DrawShaderId>` — 从 GLSL 代码到着色器 ID 的缓存

**`reset_for_live_reload()`** — 热重载时清除对象和函数缓存，但保留代码缓存，使得相同着色器无需重新编译。

---

## `DrawShaderId`（第 131–141 行）

简单的新类型包装，包含 `index: usize` 指向 `shaders` 向量。

**`false_compare_check()`** — 将索引左移 32 位生成一个 64 位哈希值，用于 `find_appendable_draw_shader_check` 快速比较。

---

## `CxDrawShader`（第 143–147 行）

着色器实例：
- `debug_id: LiveId` — 调试标识
- `os_shader_id: Option<usize>` — 平台后端着色器索引
- `mapping: CxDrawShaderMapping` — 着色器输入输出映射

---

## `DrawShaderInputs` / `DrawShaderInput`（第 149–325 行）

着色器输入布局系统，支持多种打包方式：

### `DrawShaderInputPacking`（第 156–165 行）

四种打包模式：
- `Attribute` — 顶点属性方式（整数类型需 4 槽对齐）
- `UniformsGLSLTight` — 紧凑打包，无对齐约束
- `UniformsGLSL140` — GLSL std140 规则：标量 1 槽对齐、vec2 2 槽对齐、vec3/vec4 及更大类型 4 槽对齐
- `UniformsHLSL` / `UniformsMetal` — 平台特定的打包规则

### `DrawShaderAttrFormat`（第 167–178 行）

属性格式枚举：`Float`、`UInt`、`SInt`，决定数值如何在 f32 中编解码。

### `DrawShaderInput`（第 180–188 行）

单个输入字段描述：
- `id: LiveId` — 字段名称
- `offset: usize` — 在总缓冲区中的偏移（以 f32 为单位）
- `slots: usize` — 占用的 f32 槽位数
- `attr_format: DrawShaderAttrFormat` — 属性格式

### `uniform_packing()`（第 190–210 行）

根据目标平台选择默认 uniform 打包方式：
- WebAssembly → `UniformsGLSL140`
- Android/Linux → `UniformsGLSL140`
- macOS/iOS → `UniformsMetal`
- Windows → `UniformsHLSL`

### `DrawShaderInputs::push()`（第 221–309 行）

根据打包方式将一个新输入压入布局：
- **`Attribute`**：若为整数类型且 `slots > 1`，在压入前后对齐到 4 槽边界。
- **`UniformsGLSLTight`**：直接追加，无对齐。
- **`UniformsGLSL140`**：根据 `slots` 选择对齐系数（1→1，2→2，≥3→4），对齐后压入。
- **`UniformsHLSL`**：若 `slots > 4` 则 4 槽对齐；若当前偏移的 4 槽对齐会导致跨边界则前对齐。
- **`UniformsMetal`**：与 GLSL140 类似，但 triple（3 槽）占用 4 槽物理空间。

### `DrawShaderInputs::finalize()`（第 312–324 行）

对 GLSL/HLSL/Metal 三种打包方式，在结束时对齐到 4 槽边界。

---

## `DrawShaderTextureInput` / `DrawShaderUniformBufferInput`（第 327–341 行）

纹理输入包含 `id` 和 `tex_type`（纹理类型）。Uniform 缓冲区输入包含 `id`、`block_name`（块名）、`ty`（脚本 POD 类型）、`size`（大小）、`align`（对齐）、`buffer_index`（绑定点索引）。

---

## `DrawShaderFlags`（第 343–351 行）

着色器标记：`debug_draw`（调试绘制）、`debug_layout`（调试布局）、`debug_code`（调试代码）、`draw_call_nocompare`（禁止合并）、`draw_call_always`（始终绘制）、`async_compile`（异步编译）。

---

## `CxDrawShaderCode`（第 353–357 行）

着色器代码的两种存储形式：`Separate`（顶点+片段分离）或 `Combined`（合并的代码字符串）。

---

## `CxDrawShaderMapping`（第 359–384 行）

着色器映射的核心结构，将脚本着色器定义映射到 GPU 可消费的布局：

- `source`: 脚本对象引用
- `code`: 着色器 GLSL 代码
- `flags`: 标记位
- `instances`: 完整实例布局（DynInstance + RustInstance）
- `dyn_instances`: 仅动态实例部分
- `dyn_uniforms`: 动态 uniform 布局
- `geometries`: 顶点缓冲区布局
- `textures`: 纹理输入列表
- `uniform_buffers`: uniform 缓冲区输入列表
- `samplers`: 采样器列表
- `texture_sampler_indices`: 纹理到采样器的索引映射
- `uses_time`: 是否使用 `draw_pass->time`（需要每帧重绘）
- `rect_pos` / `rect_size` / `draw_clip`: 特殊字段偏移
- `uniform_buffer_bindings`: uniform 缓冲区绑定信息
- `scope_uniforms`: 作用域 uniform 布局
- `scope_uniform_sources`: 作用域 uniform 的来源（脚本对象+键）
- `scope_uniforms_buf`: 作用域 uniform 的缓冲数据
- `geometry_id`: 关联的属性几何体 ID
- `varying_total_slots`: 后端编译后设置的 varying 总槽数

---

## `CxDrawShaderMapping` 方法

### `attr_format_from_pod_type()`（第 387–406 行）

根据脚本 POD 类型推断 GPU 属性格式：
- `U32` / `AtomicU32` / `Bool` / `Vec*u` / `Vec*b` → `UInt`
- `I32` / `AtomicI32` / `Vec*i` → `SInt`
- 其他 → `Float`

### `debug_dump_shader_draw_call()`（第 408–483 行）

调试辅助函数，输出单个绘制调用的详细状态：着色器 ID、实例数、动态 uniform 槽数、每个 uniform 字段的值、每个实例字段的值。输出的格式是分行的文本日志。

### `from_shader_output()`（第 485–831 行）

**核心函数**，从 `ShaderOutput` 构建完整的着色器映射：

1. 从脚本堆中读取调试和编译标记（`debug_draw`、`debug_layout`、`debug_code`、`async_compile`）。
2. 创建五种输入布局实例：
   - `instances`：使用 `Attribute` 打包
   - `dyn_instances`：使用 `Attribute` 打包
   - `dyn_uniforms`：使用平台感知的 `uniform_packing()`
   - `geometries`：使用 `Attribute` 打包
3. 分四阶段处理 IO 输出：
   - **阶段 1**：先处理所有 `DynInstance` 字段，添加到 `instances` 和 `dyn_instances` 两个布局中。
   - **阶段 2**：处理 `RustInstance` 字段，只添加到 `instances`。同时记录 `rect_pos`、`rect_size`、`draw_clip` 的特殊偏移。
   - **阶段 3**：处理 `Uniform` 字段，添加到 `dyn_uniforms`。
   - **阶段 4**：处理 `VertexBuffer` 字段，添加到 `geometries`。
4. 处理纹理：从输出中收集纹理输入和采样器绑定索引。
5. 处理 uniform 缓冲区：跳过系统预留缓冲区（`DrawPassUniforms`、`DrawListUniforms`、`DrawCallUniforms`），记录自定义 uniform 缓冲区。若数量超过 `DRAW_CALL_UNIFORM_BUFFER_SLOTS`（2 个）则 panic。
6. 调用 `finalize()` 完成所有布局。
7. 构建作用域 uniform 布局：按 IO 顺序处理 `ScopeUniform` 字段，从 `scope_uniforms` 中查找对应的 Source，构建 `scope_uniform_sources`。
8. 分配 `scope_uniforms_buf` 缓冲区。
9. 输出调试日志（若 `debug_layout` 开启）。
10. 检测 `uses_time` 标记：在顶点或片段代码中搜索 `"draw_pass->time"` 子串。
11. 组合所有字段返回 `CxDrawShaderMapping`。

### `fill_scope_uniforms_buffer()`（第 837–861 行）

从脚本堆中读取作用域 uniform 的值并写入 `scope_uniforms_buf`：
- 对于每个作用域 uniform 输入，通过 `heap.scope_value(source_obj, key)` 查找值。
- 调用 `DrawVars::write_value_to_f32_slots()` 将脚本值转换为 f32 槽位写入缓冲区。

此函数在着色器编译时和应用时都会被调用，确保作用域 uniform 反映最新的脚本状态。
