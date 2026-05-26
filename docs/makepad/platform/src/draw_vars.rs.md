# draw_vars.rs — 绘制变量与实例数据管理

## 概述

`draw_vars.rs` 是 Makepad 绘制管线中**数据传递层**的核心。`DrawVars` 结构体在绘制调用间传递所有可变数据：实例属性、uniform 参数、纹理绑定、uniform 缓冲区绑定等。它还负责着色器的按需编译、脚本值的槽位写入、以及脏区域更新。

---

## 常量定义（第 25–28 行）

- `DRAW_CALL_DYN_UNIFORMS: usize = 256` — 动态 uniform 的最大 f32 槽位数
- `DRAW_CALL_TEXTURE_SLOTS: usize = 16` — 纹理槽位数量
- `DRAW_CALL_UNIFORM_BUFFER_SLOTS: usize = 2` — uniform 缓冲区槽位数量
- `DRAW_CALL_DYN_INSTANCES: usize = 32` — 动态实例槽位数

---

## `DrawVars`（第 30–55 行）

核心数据传递结构体，`#[repr(C)]` 布局，字段按脚本暴露分类：

- `area: Area` — 当前绘制的区域标识
- `dyn_instance_start: usize` — 动态实例在 `dyn_instances` 数组中的起始偏移
- `dyn_instance_slots: usize` — 完整实例数据的槽数（包含 `dyn_instances` + RustInstance 部分）
- `options: CxDrawShaderOptions` — 着色器选项（深度写入、Alpha 混合等）
- `append_group_id: u64` — 合并分组的键
- `draw_shader_id: Option<DrawShaderId>` — 关联的着色器 ID
- `geometry_id: Option<GeometryId>` — 关联的几何体 ID
- `dyn_uniforms: [f32; 256]` — 动态 uniform 数组
- `texture_slots: [Option<Texture>; 16]` — 纹理绑定槽
- `uniform_buffer_slots: [Option<UniformBuffer>; 2]` — uniform 缓冲区绑定槽
- `dyn_instances: [f32; 32]` — 动态实例数据数组

**`as_slice()`**（第 167–186 行）—— 从 `dyn_instances` 的 `dyn_instance_start` 位置开始返回 `&[f32]`，长度为 `dyn_instance_slots`。关键说明：实例布局有意延伸到 `dyn_instances` 数组之后，进入 `#[repr(C)]` 着色器派生结构体中 RustInstance 字段的末尾部分，通过 `std::slice::from_raw_parts` 实现零拷贝视图。

---

## `ScriptHook` 实现（第 57–127 行）

**`on_after_apply()`** 是脚本值应用后的回调，执行以下步骤：

1. **清理过期着色器缓存**：调用 `prune_stale_object_shader_cache()`，检查对象缓存的代次是否变化，若变化则清空对象→着色器 ID 映射。

2. **编译着色器**：若非默认应用且非动画应用，调用 `compile_shader()` 从脚本值编译着色器。

3. **读取选项**：从脚本对象的 `draw_call_group`、`depth_write`、`alpha_blend`、`backface_culling` 字段读取并设置到 `self.options`。支持 `as_id()` 和 `as_f64()` 两种取值方式。

4. **填充动态值**：若有有效的 `draw_shader_id`，调用 `fill_dyn_instances()` 和 `fill_dyn_uniforms()` 从脚本对象填充实例和 uniform 数据。`shallow` 标记对 eval 类应用为 true（仅读顶层，不遍历原型链）。

5. **更新动画区域**：若为动画或 eval 应用，调用 `update_instance_areas_when_in_object()` 和 `update_uniform_areas_when_in_object()`，仅更新脚本对象中已存在的属性。

---

## `DrawVars` 公有方法

### 纹理与 Uniform 缓冲区管理

- **`set_texture(slot, texture)`** — 将纹理绑定到指定槽位。
- **`empty_texture(slot)`** — 清空指定纹理槽位（设为 None）。
- **`set_uniform_buffer(slot, buf)`** — 设置 uniform 缓冲区。
- **`empty_uniform_buffer(slot)`** — 清空 uniform 缓冲区槽位。

### 区域与重绘

- **`redraw(cx)`** — 触发区域重绘。
- **`area()`** — 返回当前区域。
- **`can_instance()`** — 返回是否存在有效着色器（即可实例化）。

### `update_rect()`（第 267–297 行）

更新已提交绘制调用中所有实例的 `rect_pos` 和 `rect_size` 字段：
1. 获取着色器映射和绘制调用。
2. 在 `instances` 输入中查找 `rect_pos` 和 `rect_size` ID。
3. 将所有实例（重复 `instance_count` 次）的对应偏移写入新的位置/大小值。
4. 设置 `instance_dirty = true` 和 `paint_dirty = true`。

### `update_instance_area_value()`（第 299–324 行）

根据 LiveId 名称更新特定实例字段。在实例输入中匹配 `id[0]`，将所有实例的该字段更新为 `as_slice()` 中的当前值。

### `get_instance()`（第 326–340 行）

在 `as_slice()` 中查找指定 ID 的实例输入，将其值复制到输出数组中。

### `get_instance_on_area()`（第 344–387 行）

从已提交的绘制调用的实例缓冲区中读取指定 ID 的值。进行多层安全检查：
1. 检查 `draw_shader_id` 和 `area.valid_instance()`。
2. 检查输入的 `checked_index`（允许无效 ID）。
3. 检查 `draw_item_id` 的边界。
4. 校验 `instance_count` 不超过 `available / stride`。
5. 校验 `base + input.slots` 不超过缓冲区长度。

### `set_dyn_instance()`（第 389–403 行）

在 `dyn_instances` 数组中设置动态实例值。偏移计算为 `dyn_instances.len() - mapping.dyn_instances.total_slots + input.offset`（即从数组末尾向前推算）。

### `get_uniform()` / `set_uniform()`（第 405–433 行）

在 `dyn_uniforms` 数组中按 ID 读取或写入 uniform 值。遍历 `dyn_uniforms.inputs` 匹配 LiveId。

### `set_uniform_on_area()`（第 437–464 行）

在绘图完成后更新 uniform 值并立即同步到已提交的绘制调用：
1. 在映射中查找 uniform 输入。
2. 更新 `self.dyn_uniforms`。
3. 若存在有效区域，同步更新 `draw_call.dyn_uniforms`。
4. 设置 `uniforms_dirty` 和 `paint_dirty` 标记。

### `set_instance_on_area()`（第 468–511 行）

在绘图完成后更新实例数据并同步到绘制调用：
1. 查找实例输入。
2. 获取绘制调用和实例缓冲区。
3. 进行缓冲区边界检查（`instance_count > max_count` 时日志输出 "stale" 并跳过）。
4. 更新所有实例的对应字段。
5. 设置 `instance_dirty` 和 `paint_dirty` 标记。

---

## 内部填充方法

### `fill_dyn_instances()`（第 513–546 行）

从脚本对象填充动态实例字段：
1. 计算 `base_offset = dyn_instances.len() - mapping.dyn_instances.total_slots`（从末尾预留空间）。
2. 对每个动态实例输入，调用 `extract_shader_io_value()` 获取脚本值。
3. 若非 nil 且非错误，调用 `write_value_to_f32_slots()` 写入。

### `fill_dyn_uniforms()`（第 548–578 行）

类似上述逻辑，填充动态 uniform 字段。偏移直接使用 `input.offset`（无 base_offset，因为 uniform 从头开始排列）。

### `extract_shader_io_value()`（第 580–609 行）

从脚本对象提取着色器 IO 值：
1. 若 `shallow` 为 true，仅从对象自己的 map 中查找（原型链不遍历）。
2. 否则使用 `heap.value()` 遍历原型链。
3. 特殊处理：若值是一个 `ShaderIO` 对象且类型匹配，返回其 prototype（即实际的数值）。

### `write_value_to_f32_slots()`（第 612–905 行）

**核心类型转换函数**，将各种脚本值格式转换为 f32 槽位：

支持以下类型，按优先级顺序尝试：
1. **f64**：标量展开到所有槽位。
2. **u40**（LiveId 常用）：标量展开。
3. **f32**：标量展开。
4. **f16**：半精度转 f32 并展开。
5. **u32** / **i32**：整数展开。
6. **bool**：true→1.0，false→0.0。
7. **Color(u32 RGBA)**：调用 `Vec4f::from_u32()` 分解为 rgba 分量。
8. **`_repr_u32_enum_value` 对象**：解包枚举值。
9. **Pod 类型**（最复杂的处理）：
   - `F32` / `F16` / `U32` / `I32` / `Bool`：标量展开
   - `Vec2/3/4f`：逐分量 `f32::from_bits(data[i])`
   - `Vec2/3/4h`：半精度解码（两个分量打包在一个 u32 中）
   - `Vec2/3/4u/i/b`：整数转换
   - `Mat`（矩阵）：列主序排列
   - 其他 POD 类型：填充 0

所有类型都根据 `attr_format`（Float/UInt/SInt）进行适当的位级转换：
- `Float`：直接转为 f32
- `UInt`：用 `f32::from_bits()` 保留位模式
- `SInt`：先转为 i32 再用 `f32::from_bits()`

---

## 着色器编译（第 907–1107 行）

### `compile_shader()`（第 908–1054 行）

**核心编译入口**，将脚本对象编译为平台着色器代码：

1. **缓存查找**：
   - 首先查 `cache_object_id_to_shader`（对象 ID → 着色器）。
   - 然后计算函数哈希，查 `cache_functions_to_shader`（函数哈希 → 着色器）。
   - 命中缓存则调用 `finalize_cached_shader()` 快速配置。

2. **ShaderOutput 设置**：
   - 设为 `ShaderBackend::Glsl`，`use_vulkan = false`。
   - 调用 `pre_collect_rust_instance_io()` 预收集 Rust 侧实例定义。
   - 调用 `pre_collect_shader_io()` 预收集着色器 IO。

3. **编译顶点着色器**：
   - 从脚本对象查找 `vertex` 函数。
   - 调用 `ShaderFnCompiler::compile_shader_def()` 生成 ShaderOutput。

4. **编译片段着色器**：
   - 类似方式编译 `fragment` 函数。

5. **代码生成**：
   - `assign_uniform_buffer_indices()` 分配绑定点。
   - `create_struct_defs()` 生成结构定义。
   - `glsl_create_vertex_shader()` / `glsl_create_fragment_shader()` 生成完整的 GLSL 源码。

6. **缓存和注册**：
   - 尝试 `cache_code_to_shader` 缓存（相同源码避免重复编译）。
   - 若未命中，创建新的 `CxDrawShader` 实例。
   - 填充 `dyn_instance_start` 和 `dyn_instance_slots`。
   - 插入三条缓存条目：对象 ID、函数哈希、源码哈希。
   - 将索引加入 `compile_set`，通知后端编译。

### `compute_shader_functions_hash()`（第 1058–1090 行）

计算着色器函数的哈希值，用于缓存匹配：
1. 从当前对象开始，遍历原型链。
2. 对每个对象的 map 条目，检查值是否为函数对象。
3. 若是，将方法名和 `ScriptFnPtr::Script` 的 IP 地址哈希到 `LiveId` 中。
4. 返回最终的哈希值。

### `finalize_cached_shader()`（第 1094–1107 行）

缓存命中时的快速配置方法：
1. 从缓存中读取映射的 `dyn_instances` 和 `instances` 槽数。
2. 设置 `dyn_instance_start` 和 `dyn_instance_slots`。
3. 设置 `draw_shader_id`。
4. 从映射中读取 `geometry_id`（无需重新编译）。
