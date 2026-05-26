# `headless/shader.rs` — 无头着色器编译与代码生成

## 概述

本文件实现了 Makepad 着色器 DSL 到 Rust JIT 模块的完整编译链路。它负责：解析着色器定义 → 编译顶点/片段函数 → 生成包含 `RenderCx` 结构和入口点的 Rust 源码 → 生成模块布局导出函数。

---

## 着色器编译入口 — `DrawVars::compile_shader()`

此方法在 Makepad 脚本 VM 中每次遇到着色器对象时被调用：

1. **缓存查找**（三重缓存）：
   - 按 `object_id` 查找 → 命中直接使用缓存着色器
   - 按函数哈希 `fnhash` 查找 → 命中后更新 `object_id` 缓存
   - 按代码哈希 `code` 查找 → 命中后更新前两个缓存

2. **编译新着色器**：
   - 设置 `ShaderBackend::Rust` 和 `use_vulkan = false`
   - 调用 `pre_collect_rust_instance_io()` 和 `pre_collect_shader_io()` 收集 IO
   - 编译 `vertex` 函数（`ShaderMode::Vertex`）
   - 编译 `fragment` 函数（`ShaderMode::Fragment`）

3. **源码生成**（非 no_draw 模式）：
   - 调用 `generate_headless_rust_shader_module()` 生成完整 Rust 源码
   - 提取 `varying_total_slots`

4. **创建 `CxDrawShader`**：
   - 从 `ShaderOutput` 构建 `CxDrawShaderMapping`
   - 填充 `scope_uniforms_buffer`
   - 将着色器 ID 加入 `compile_set`，等待后续 JIT 编译

---

## `headless_compile_shaders()` — JIT 编译所有待编译着色器

在 `Cx` 上实现，遍历 `compile_set`：

1. **no_draw 模式**: 直接创建标记为加载错误的空 `CxOsDrawShader`
2. **正常模式**:
   - 从 `CxDrawShaderCode::Combined` 获取源码
   - 计算 `source_hash`
   - 去重：如果已有相同 `source_hash` 的 `CxOsDrawShader`，复用
   - 调用 `self.os.shader_jit.compile_and_load(source_hash, source)`
   - 从 JIT 模块查询布局信息：
     - `makepad_headless_render_cx_size` — RenderCx 总大小
     - `makepad_headless_rcx_vary_offset` — varying 区域偏移
     - `makepad_headless_rcx_quad_mode_offset` — quad_mode 偏移
     - `makepad_headless_flat_varying_slots` — 非插值槽位
     - `makepad_headless_uses_derivatives` — 是否使用导数
     - `makepad_headless_rcx_frag_offset` — 片段输出偏移
     - `makepad_headless_rcx_discard_offset` — discard 偏移
   - 设置后向兼容回退：无导数导出时默认为 `true`

---

## 代码生成 — `generate_headless_rust_shader_module()`

生成完整的 Rust 源码文件，包含：

### 生成的文件结构

```
//! Auto-generated Makepad headless shader module.
#![allow(unused_variables, unused_mut, ...)]

// [1] 运行时前置代码（shader_runtime_preamble.rs）
//     包含 Vec2f/3f/4f, Mat4f, Texture2D, Sdf2d, 数学函数等

// [2] 用户自定义结构体定义（过滤掉 preamble 已有的类型）

// [3] RenderCx 结构体
//     #[repr(C)] 纯 POD，分组排列

// [4] dFdx/dFdy 回退函数（内联由编译器生成）

// [5] 着色器函数（io_vertex, io_fragment 等）

// [6] 导出函数：
//     - makepad_headless_shader_version() → u32
//     - makepad_headless_flat_varying_slots() → u32
//     - makepad_headless_uses_derivatives() → u32

// [7] RenderCx 布局导出函数（偏移量查询）

// [8] makepad_headless_fill_rcx(...) — 填充 uniform/texture

// [9] makepad_headless_vertex(...) — 顶点着色器入口

// [10] makepad_headless_fragment(...) — 片段着色器入口
```

### RenderCx 结构体布局（`write_render_cx_struct`）

`#[repr(C)]` 纯 POD 结构体，按区域分组以便宿主端按偏移写入：

```
Group 1:  Varyings         dyninst_*, rustinst_*, var_*
Group 1b: Quad derivatives  quad_mode, quad_slot, quad_lane_x/y, quad_dx/dy_buf
Group 3:  Uniforms          uni_*, su_*
Group 4:  Uniform buffers   unibuf_*
Group 5:  Textures          tex_* (Texture2D POD)
Group 6:  Geometry          vb_* (仅顶点着色器)
Group 7:  Position          vtx_pos (仅顶点着色器)
Group 8:  Fragment output   frag_fb0 (仅片段着色器)
Group 9:  Discard flag      discard
```

### 布局导出函数（`write_render_cx_layout_exports`）

生成的导出函数使用指针算术计算偏移：
```rust
#[no_mangle]
pub extern "C" fn makepad_headless_rcx_vary_offset() -> u32 {
    let base = std::ptr::null::<RenderCx>();
    unsafe { (std::ptr::addr_of!((*base).dyninst_xxx) as usize) as u32 }
}
```

### 顶点着色器入口（`write_vertex_entry`）

1. 从 `geom_ptr` 构建切片 → 读取到 `rcx.vb_*`
2. 从 `inst_ptr` 构建切片 → 读取到 `rcx.dyninst_*` / `rcx.rustinst_*`
3. 解包 uniform → `rcx.unibuf_*` / `rcx.uni_*` / `rcx.su_*`
4. 调用 `io_vertex(&mut rcx)`
5. 写入 `*out_pos = [vtx_pos.x, vtx_pos.y, vtx_pos.z, vtx_pos.w]`
6. 将 varyings 打包输出到 `varying_out` 缓冲区

### 片段着色器入口（`write_fragment_entry`）

1. 验证 `rcx_f32s ≥ size_of::<RenderCx>() / size_of::<f32>()`
2. 零拷贝转换：`let rcx = &mut *(rcx_ptr as *mut RenderCx);`
3. 重置输出字段：`discard = 0.0`, `frag_fb0 = Vec4f::zero()`
4. 调用 `io_fragment(rcx)`
5. 返回 `discard != 0 ? 0 : 1`

### 填充 Rcx 入口（`write_fill_rcx_entry`）

1. 解包 uniform buffer 变量（从 `uniform_ptrs`/`uniform_lens` 索引数组）
2. 解包动态 uniform（按 `DrawShaderInputPacking` 布局解析）
3. 解包 scope uniform
4. 填充 texture 字段（从 `tex_infos_ptr` 数组读取 `[data_ptr, data_len, width, height]`）

---

## 辅助函数

| 函数 | 说明 |
|------|------|
| `count_varying_slots()` | 计算 varying 总 f32 槽位（dyn_inst + rust_inst + varyings）|
| `count_flat_varying_slots()` | 计算非插值 f32 槽位（dyn_inst + rust_inst）|
| `write_varying_fields()` | 生成 varying 结构体字段声明（支持前缀）|
| `write_dfdx_dfdy_fallbacks()` | 生成 dFdx/dFdy 回退函数 |
| `write_shader_functions()` | 生成 shader 函数体（带 unsafe 包装）|
| `write_filtered_struct_defs()` | 生成结构体定义（过滤 preamble 已有类型）|
| `write_static_assign()` / `write_static_assign_typed()` | 从 f32 切片读取值赋值给变量 |
| `write_static_pack_typed()` | 将变量值写入 f32 切片 |
| `write_uniform_unpack()` | 生成 uniform 解包代码 |
| `headless_uniform_packing()` | 返回当前平台的 uniform 打包策略 |
| `hash_string()` | 计算着色器源码的 64 位哈希 |
