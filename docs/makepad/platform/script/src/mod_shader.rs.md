# `mod_shader.rs` — 着色器模块注册

## 概述

本文件定义了 Splash 脚本与 GPU 着色器系统之间的类型映射和注册逻辑。它通过 `define_shader_module` 函数在 `mod.shader` 命名空间下注册了 20+ 种着色器 I/O 类型（实例数据、uniform、纹理、缓冲区、varying 等）的构造器，并提供两个测试编译方法用于验证着色器管线的代码生成正确性。

---

## 着色器 I/O 类型常量

`ShaderIoType(u32)` 是一个新类型包装，用于标识着色器管线中不同种类的 I/O 数据。以下是完整常量表（按序号分组）：

| 常量 | 序号 | 用途 |
|------|------|------|
| `SHADER_IO_RUST_INSTANCE` | 0 | Rust 侧直接管理的实例数据 |
| `SHADER_IO_DYN_INSTANCE` | 1 | 脚本侧动态实例数据 |
| `SHADER_IO_DYN_UNIFORM` | 2 | 脚本侧动态 uniform 数据 |
| `SHADER_IO_UNIFORM_BUFFER` | 3 | Uniform 缓冲区（常量缓冲区） |
| `SHADER_IO_VERTEX_BUFFER` | 4 | 顶点缓冲区 |
| `SHADER_IO_VARYING` | 5 | 顶点→片元传递的可变数据 |
| `SHADER_IO_VERTEX_POSITION` | 6 | 顶点位置输入 |
| `SHADER_IO_TEXTURE_1D` ~ `SHADER_IO_TEXTURE_DEPTH_ARRAY` | 7–17 | 各种纹理类型 |
| `SHADER_IO_SAMPLER` | 18 | 采样器对象 |
| `SHADER_IO_BUFFER_R/W/RW` | 19–21 | 存储缓冲区（只读/写/读写） |
| `SHADER_IO_SCOPE_UNIFORM` | 22 | 作用域 uniform（DrawCall 级共享） |
| `SHADER_IO_FRAGMENT_OUTPUT_0` ~ `+7` | 23–30 | 片元输出（多渲染目标） |

---

## `define_shader_module(heap, native)`

### 模块创建

- 调用 `heap.new_module(id!(shader))` 创建 `mod.shader` 模块对象。
- 随后通过 `native.add_method` 逐条注册每个 I/O 类型的构造器。

---

## 统一 I/O 构造器模式

除 `depth_clip`、`fragment_output`、`vertex_buffer`、`test_compile_draw` 和 `test_compile_draw_contains` 外，所有构造器遵循相同的实现模式：

1. **参数**: `(value = NIL)` — 接受一个脚本值作为原型。
2. **逻辑**:
   - 用 `script_value!` 取出参数值。
   - `heap.new_with_proto(value)` 创建一个继承自 `value` 的新脚本对象。
   - `heap.set_shader_io(obj, SHADER_IO_XXX)` 在新对象上设置着色器 I/O 类型的元数据标记。
   - 返回 `obj.into()`。

**示例**: `shader.texture_2d(value)` 创建一个标记为 `SHADER_IO_TEXTURE_2D` 的对象，该对象继承自 `value` 的所有属性，下游的着色器编译器通过 `shader_io` 标记来识别它应该被绑定到哪个 GPU 槽位。

---

## 特殊构造器

### `shader.depth_clip(world, color, clip) → ScriptValue`

**实现逻辑**:
1. 接收三个参数：`world`（可选）、`color`（可选）和 `clip`（默认 0.0）。
2. 直接从参数包中取出 `color` 键对应的值返回——这是一个简单的访问器，用于从深度裁剪参数包中提取颜色值。
3. 返回 `color` 参数值。

### `shader.vertex_buffer(value, buf) → ScriptValue`

**实现逻辑**:
1. 接收两个参数：`value`（类型原型）和 `buf`（缓冲区对象）。
2. 创建新对象并设置 `SHADER_IO_VERTEX_BUFFER` 标记。
3. 额外使用 `set_script_value!` 将 `buf` 写入新对象的 `buffer` 字段，供下游着色器编译器获取实际的顶点缓冲区数据。

### `shader.fragment_output(index, ty) → ScriptValue`

**实现逻辑**:
1. 接收 `index`（渲染目标索引，默认 NIL）和 `ty`（输出类型原型，默认 NIL）。
2. 将 `index` 转换为无符号整数并限制在 `[0, 7]` 范围内（最大 8 个 MRT）。
3. 使用 `SHADER_IO_FRAGMENT_OUTPUT_0.0 + index` 计算偏移后的 I/O 类型。
4. 设置标记后返回新对象。

---

## 测试编译方法

### `shader.test_compile_draw(io_self) → NIL`

**实现逻辑**:
1. 从 `io_self` 参数提取脚本对象引用。
2. 创建 `ShaderOutput` 默认实例，设置 `backend = ShaderBackend::Metal`、`use_vulkan = false`。
3. 分两步预收集 I/O:
   - `pre_collect_rust_instance_io`: 从 `io_self` 中提取 Rust 侧实例数据标记。
   - `pre_collect_shader_io`: 收集所有脚本侧标记的 I/O 条目。
4. 在 `io_self` 中查找名为 `vertex` 的方法对象。如果存在：
   - 设置 `output.mode = ShaderMode::Vertex`。
   - 调用 `ShaderFnCompiler::compile_shader_def` 编译顶点着色器定义，使用 `NoTrap`（入口点无脚本参数校验）。
5. 同样查找 `fragment` 方法并编译。
6. `output.assign_uniform_buffer_indices(&vm.bx.heap, 3)` 分配 uniform 缓冲区索引（最多 3 个）。
7. 依次调用 Metal 后端代码生成方法，生成完整的 Metal 着色器源码到 `out` 字符串：
   - `create_struct_defs`: 结构体定义
   - `metal_create_instance_struct`: 实例数据结构
   - `metal_create_uniform_struct`: uniform 常量缓冲区结构
   - `metal_create_scope_uniform_struct`: 作用域 uniform 结构
   - `metal_create_io_struct`: 输入输出接口结构
   - `metal_create_varying_struct`: varying 结构
   - `metal_create_vertex_buffer_struct`: 顶点缓冲结构
   - `metal_create_sampler_decls`: 采样器声明
   - `metal_create_helpers`: 辅助函数
   - `metal_create_io_vertex_struct`, `metal_create_vertex_fn`, `metal_create_io_fragment_struct`
8. 生成的源码当前被注释掉（不会打印到控制台），该函数只用于验证编译过程不报错。

### `shader.test_compile_draw_contains(io_self, needle) → bool`

**实现逻辑**:
1. 执行与 `test_compile_draw` 相同的完整编译管线（从 I/O 收集到 Metal 代码生成）。
2. 额外增加了 `create_functions` 和 `metal_create_io_framebuffer_struct`/`metal_create_fragment_main_fn` 调用，生成更完整的片元着色器框架。
3. 将 `needle` 参数解释为字符串，检查最终生成的 `out` 代码中是否包含该子串。
4. 返回布尔值 `out.contains(&needle)`。

**用途**: 用于着色器编译器测试——验证某个结构名、变量名或宏是否被正确生成到 Metal 源码中。

---

## 设计要点

- **类型标记分离**: I/O 类型构造器（`texture_2d`、`uniform` 等）和着色器编译逻辑完全分离。前者只负责创建带标记的对象，后者（`ShaderFnCompiler`、`ShaderOutput`）在编译阶段才解析这些标记。
- **分层后端设计**: 尽管测试编译目前固定为 Metal 后端，`ShaderBackend` 枚举支持扩展（Vulkan、GLSL、HLSL、WGSL），`pre_collect_*` 和编译流程是后端无关的。
- **测试管线完整性**: `test_compile_draw_contains` 生成的代码是最完整的管线（包括 `create_functions` 和 `metal_create_fragment_main_fn`），适合作为集成测试的标准入口。
- **NoTrap 策略**: 入口点着色器函数（`vertex`、`fragment`）使用 `NoTrap` 而非 `Trap` 进行编译，因为它们是编译期已知的固定签名的函数，不需要脚本运行时参数验证。
