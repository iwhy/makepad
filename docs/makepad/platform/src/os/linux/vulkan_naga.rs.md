# vulkan_naga.rs

One-liner (EN): WGSL-to-SPIR-V shader compilation using the naga library for the Vulkan rendering backend.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/vulkan_naga.rs` (145 行)
- **核心作用**: 使用 `naga` crate 将 Makepad 的 WGSL 着色器源代码编译为 SPIR-V 二进制格式（Vertex + Fragment），供 Vulkan 后端使用。

## 关键类型

### `CxVulkanShaderBinary`

| 字段 | 类型 | 说明 |
|------|------|------|
| `vertex_spirv` | `Option<Vec<u32>>` | 顶点着色器 SPIR-V 二进制 |
| `fragment_spirv` | `Option<Vec<u32>>` | 片段着色器 SPIR-V 二进制 |
| `dyn_uniform_binding` | `u32` | 动态 uniform 的绑定编号 |
| `texture_binding_base` | `u32` | 纹理绑定的起始编号 |
| `sampler_binding_base` | `u32` | 采样器绑定的起始编号 |
| `xr_depth_binding` | `u32` | XR 深度绑定的编号 |
| `geometry_slots` | `usize` | 几何体槽位数量 |
| `instance_slots` | `usize` | 实例槽位数量 |

## 关键函数

### `compile_wgsl_to_spirv(wgsl: &str) -> Result<(Option<Vec<u32>>, Option<Vec<u32>>), String>`

内部函数，将 WGSL 源代码编译为 SPIR-V。

**流程**:
1. **解析**: `naga::front::wgsl::parse_str(wgsl)` — 将 WGSL 文本解析为 Naga IR 模块
2. **验证**: `valid::Validator::validate(&module)` — 使用所有验证标志和所有能力
3. **检查入口点**: 查找 `vertex_main` 和 `fragment_main` 入口点
4. **生成 SPIR-V**: `spv::write_vec()` — 分别输出顶点和片段着色器的 SPIR-V

**错误处理**: WGSL 解析错误包含带行号上下文的格式化信息，使用 `extract_error_line` 和 `wgsl_context` 辅助函数生成诊断信息。

### `compile_draw_shader_wgsl_to_spirv(vm, io_self, layout_source, xr_multiview) -> Result<CxVulkanShaderBinary, String>`

公开函数，完整着色器编译管线。

**流程**:
1. 调用 `compile_draw_shader_wgsl_source` 生成 WGSL 源码和绑定信息
2. 若设定了 `MAKEPAD_DUMP_VULKAN_WGSL` 环境变量，记录生成的 WGSL
3. 调用 `compile_wgsl_to_spirv` 编译为 SPIR-V
4. 将绑定信息（dyn_uniform, texture, sampler, xr_depth 绑定编号 + geometry/instance slots）打包为 `CxVulkanShaderBinary`

## 辅助函数

| 函数 | 说明 |
|------|------|
| `extract_error_line(details) -> Option<usize>` | 从 naga 错误消息中提取行号（`wgsl:` 标记后） |
| `wgsl_context(wgsl, line, radius) -> String` | 生成错误行周围的上下文代码片段 |

## SPIR-V 编译选项

```rust
spv::Options {
    lang_version: (1, 3),           // SPIR-V 1.3
    flags: spv::WriterFlags::empty(),
    fake_missing_bindings: true,     // 为缺失绑定生成虚拟条目
    zero_initialize_workgroup_memory: None,  // 不清零工作组内存
    bounds_check_policies: default,
    debug_info: None,               // 不生成调试信息
}
```

## 实现细节
- 仅支持 SPIR-V 1.3 输出
- `fake_missing_bindings: true` 确保即使某些绑定在着色器中未引用也能生成有效 SPIR-V
- `vertex_main` / `fragment_main` 是固定的入口点名称
- 若 `MAKEPAD_DUMP_VULKAN_WGSL=1`，可在编译失败时查看生成的 WGSL 以辅助调试
