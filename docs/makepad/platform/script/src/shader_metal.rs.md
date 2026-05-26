# `shader_metal.rs` — Apple Metal MSL 后端代码生成器

## 概述

本文件实现了 Makepad 着色器编译器的 Metal Shading Language（MSL）后端，将中间表示翻译为 Metal 2.0+ 兼容着色器源码。MSL 后端的设计原则是使用**指针和间接引用**来减少数据拷贝，并通过 `[[attribute(N)]]` 语法将着色器资源绑定到 Metal 缓冲区/纹理/采样器槽位。

**核心特点：**
- 使用 `constant Type *` 指针传递 uniform 和顶点缓冲数据（避免值拷贝）
- 使用 `thread Type *` 传递实例数据
- 通过 `[[buffer(N)]]`、`[[texture(N)]]`、`[[sampler(N)]]` 属性绑定资源槽位
- 实例数据使用 `IoInstanceRaw`（packed 布局）和 `IoInstance`（对齐布局）双层结构，由 `_mp_decode_instance` 函数在 GPU 上转换
- 内置 4x4 矩阵求逆辅助函数 `_mp_inverse`
- 通过 `constexpr sampler` 声明编译期常量采样器

---

## 公有方法

### `metal_create_helpers(&self, out)`

生成 MSL 辅助函数。当前包含 `_mp_inverse` — 一个手动展开的 4x4 矩阵求逆函数，使用余子式（cofactor）展开法计算。包含零行列式检测并返回单位矩阵作为回退。此实现避免了 Metal 标准库中可能缺失的 `inverse` 内置函数（某些旧版本或特定设备上不可用）。

### `metal_create_io_struct(&self, vm, out)`

生成 `Io` 结构体，这是整个着色器的统一上下文引用。包含：

- `constant IoUniform *u` — uniform 常量数据指针
- `thread IoInstance *i` — 实例数据指针
- `constant IoScopeUniform *su` — 作用域 uniform 指针（仅在存在 scope uniform 时）
- 所有纹理（`textureNd<float>`）、采样器（`sampler`）成员
- 所有 UniformBuffer 的 `constant Type *u_{name}` 指针
- `constant IoVertexBuffer *vb` — 顶点缓冲指针

### `metal_create_scope_uniform_struct(&self, vm, out)`

生成 `IoScopeUniform` 结构体，包含所有 `ScopeUniform` 字段。仅在存在 scope uniform 时生成。这些数据由 CPU 在绘制前从脚本作用域读取并写入 GPU 缓冲区。

### `metal_create_instance_struct(&self, vm, out)`

这是 Metal 后端最有特色的部分，生成三层结构：

1. **`IoInstanceRaw`** — 使用 packed（紧凑对齐）类型的原始实例结构体，与 CPU 侧 `repr(C)` 结构体的内存布局精确匹配。包含所有动态实例和 Rust 实例字段。特别地，`Mat4x4f` 被拆分为 4 个 `packed_float4` 列，与 CPU 的列优先布局完全一致。

2. **`IoInstance`** — 使用常规对齐（simd 对齐）的实例结构体，用于着色器函数内的方便访问。

3. **`_mp_decode_instance`** — 内联转换函数，将 `IoInstanceRaw` 转换为 `IoInstance`。对 `Mat4x4f` 字段执行 `float4x4(float4(raw.col0), float4(raw.col1), ...)` 构造；其他字段直接拷贝。

这种双层设计的原因是 Metal 的 packed 类型和标准类型有不同的对齐要求，CPU 传递的裸二进制数据需要转换为 Metal 可高效读取的对齐格式。

### `metal_create_uniform_struct(&self, vm, out)`

生成 `IoUniform` 结构体，包含所有 `Uniform` 类型的 IO 字段。用作顶点和片元着色器函数签名的 `constant IoUniform *u` 参数的对应类型。

### `metal_create_varying_struct(&self, vm, out)`

生成 `IoVarying` 结构体，包含顶点到片元传递的数据：

- `_iid` — 实例 ID（使用 `[[flat]]` 限定符防止插值），放在结构体最前面以保证偏移量一致性
- 所有 `Varying` 类型的 IO 字段
- `_position` — 裁剪空间位置（使用 `[[position]]` 属性标记），必须是 `float4`

注意：与 HLSL 不同，Metal 的 varying 不传递实例字段。实例数据通过 `buffer(1)` 的 `_mp_decode_instance` 在片元阶段按需读取。

### `metal_create_vertex_buffer_struct(&self, vm, out)`

生成 `IoVertexBuffer` 结构体，包含所有 `VertexBuffer` 类型的字段。使用 packed 类型（`pod_type_name_packed_from_ty`）以匹配 CPU 侧的紧凑布局。

### `metal_create_io_vertex_struct(&self, _vm, out)`

生成 `IoV` 结构体，顶点着色器的轻量级上下文：

- `thread IoVarying *v` — 指向本地 varying 结构体的指针
- `vid` — 顶点 ID
- `iid` — 实例 ID

### `metal_create_vertex_fn(&self, vm, out)`

生成 `vertex vertex_main` 函数，Metal 顶点着色器的入口点。函数签名为：

```metal
vertex IoVarying vertex_main(
    constant IoVertexBuffer *vb [[buffer(0)]],
    constant IoInstanceRaw *i_raw [[buffer(1)]],
    constant IoUniform *u [[buffer(2)]],
    [UniformBuffer parameters with [[buffer(N)]]],
    [IoScopeUniform *su [[buffer(N)]] if needed],
    [Texture parameters with [[texture(N)]]],
    [Sampler parameters with [[sampler(N)]]],
    uint vid [[vertex_id]],
    uint iid [[instance_id]]
)
```

函数体：

1. 创建 _io`上下文并填充指针
2. 调用 `_mp_decode_instance(i_raw[iid])` 解码当前实例的数据
3. 将各缓冲区/纹理指针赋值到 `_io`
4. 创建局部 `IoVarying` 并在栈上初始化
5. 创建 `IoV` 指针结构体指向局部 varying
6. 设置 `vid`、`iid` 和 `_iid`
7. 调用 `io_vertex(_io, _iov)`（若返回 `Vec4f` 则赋给 `_position`）
8. 确保 instance ID 在用户代码后重新设置
9. 返回 `_v`（IoVarying）用于光栅化

UniformBuffer 使用 `assign_uniform_buffer_indices` 分配的索引，从 buffer(3) 开始。

### `metal_create_fragment_main_fn(&self, vm, out)`

生成 `fragment IoFb fragment_main` 函数，Metal 片元着色器的入口点。函数签名与顶点版本对称：

1. `[[stage_in]]` 接收光栅化后的 `IoVarying`（由顶点着色器输出插值而来）
2. 重新声明所有 `[[buffer(N)]]`、`[[texture(N)]]`、`[[sampler(N)]]` 参数
3. 使用 `v._iid` 索引 `i_raw` 来解码当前实例（片元阶段需要实例数据时必须通过这种间接方式获取）
4. 创建 `IoFb` 和 `IoF` 结构体
5. `IoF.fb` 指向 `_iofb`（片元输出）
6. 调用 `io_fragment(_io, _iof)`
7. 返回 `_iofb`

### `metal_create_io_fragment_struct(&self, _vm, out)`

生成 `IoF` 结构体，片元着色器的轻量级上下文：

- `thread IoVarying *v` — 指向插值后 varying 的指针
- `thread IoFb *fb` — 指向片元输出结构体的指针

### `metal_create_io_framebuffer_struct(&self, vm, out)`

生成 `IoFb` 结构体，片元输出帧缓冲包装。包含所有 `FragmentOutput` 字段，每个字段使用 `[[color(N)]]` 属性绑定到对应的渲染目标附件索引。

### `metal_create_sampler_decls(&self, out)`

生成 `constexpr sampler` 声明，Metal 编译期常量采样器对象。每个采样器包含：

- `filter::nearest` 或 `filter::linear`
- `mip_filter::linear`
- `address::repeat` / `clamp_to_edge` / `clamp_to_zero` / `mirrored_repeat`
- `coord::normalized` 或 `coord::pixel`

使用 `constexpr` 关键字使采样器配置在编译期固定，消除运行时开销。
