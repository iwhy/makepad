# shader_vars.rs — 变量/字段操作编译

**源码路径:** `platform/script/src/shader_vars.rs`  
**总行数:** 1542 行  
**核心职责:** 将 Splash 字节码中的变量声明（let/var）、赋值操作、字段访问、数组索引、Shader IO 字段访问等翻译为目标后端着色器代码。

---

## 核心架构

此文件是着色器编译器的**变量和字段访问中枢**。最复杂的逻辑集中在 `handle_field` 方法（最长的方法之一），它需要根据实例类型（Pod、PodPtr、IoSelf、ScopeObject、ScopeUniformBuffer、Texture 等）和字段性质选择不同的代码生成策略。

Shader IO（输入/输出）体系：
- 每个 IO 字段（`shader.uniform`、`shader.varying`、`shader.instance`、纹理等）在编译时被收集到 `output.io` 向量中
- 字段访问时生成带有特定前缀的表达式（如 `_uniform_`、`_instance_`、`_varying_`），前缀由后端根据 `ShaderMode` 和 `ShaderIoType` 确定
- 对于未显式标记的字段，通过 `infer_unmarked_shader_io_type` 进行隐式推断（标量→实例，向量/矩阵→uniform）

---

## 方法详解

### `handle_log` — 着色器调试日志（第20-34行）

从栈顶 `peek` 值和类型，调用 `crate::makepad_error_log::log_with_level` 输出 `value:type_name` 格式的日志。使用 `ip_to_loc` 获取源代码位置信息。

### `shader_type_to_string` — ShaderType 转可读字符串（第36-78行）

将 `ShaderType` 枚举的各个变体转换为人类可读的字符串表示。对 `Pod`/`PodType`/`PodPtr` 使用 `pod_type_name` 获取类型名。对 `Id` 类型尝试在 `shader_scope` 中解析以获取实际 Pod 类型名。对 `Range` 输出 `range<TypeName>` 格式。

### `handle_assign` — 变量赋值（第80-124行）

弹出值和目标 ID，在 `shader_scope` 中查找变量。确保目标是可变的 `var`（不是不可变的 `let` 或 `Param`）。生成 `var_name = value` 字符串，将结果类型设为 `void`。变量名通过 `backend.map_local_name` 或 `backend.map_param_name` 映射。

### `handle_assign_field` — 字段赋值（第126-344行）

处理 `instance.field = value` 形式的赋值。从栈上依次弹出：值（已决议）、字段 ID、实例表达式（已决议）。根据实例类型分派：

1. **`ShaderType::Pod`**（第132-165行）：查找字段类型并验证值类型兼容。对向量类型使用 `map_field_name_typed`（可能使用 `.x`/`.r` 等 swizzle 语法）。生成 `instance.field = value`。

2. **`ShaderType::PodPtr`**（第179-226行）：指针类型（Metal uniform buffer），使用 `->` 语法：`ptr->field = value`。

3. **`ShaderType::IoSelf`**（第227-324行）：Shader IO 字段赋值。验证赋值权限（只允许在合适的 shader 模式下赋值——Vertex 模式可写 varying/vertex_position，Fragment 模式可写 fragment_output）。将 IO 字段注册到 `output.io`（如果尚未注册）。根据 `ShaderIoPrefix`（Prefix/Full/FullOwned）生成 IO 字段访问表达式。

### `handle_array_index` — 数组索引（第346-398行）

弹出索引和数组实例，通过 `type_table_elem_type` 确定元素类型。验证索引类型是整数。生成 `instance[index]` 表达式。元素类型被推回栈上作为表达式的结果类型。

### `handle_assign_index` — 索引赋值（第400-465行）

处理 `instance[index] = value` 形式的数组索引赋值。与 `handle_array_index` 类似但生成完整的赋值表达式，结果类型为 void。

### `handle_assign_me` — 命名参数收集（第467-497行）

用于构造函数的命名参数收集。弹出 ID 和值，在 ME 栈顶查找 `ShaderMe::Pod` 标记，将参数作为 `ShaderPodArg { name: Some(id), ty, s: value }` 推入 args 向量。确保不混用命名和位置参数。

### `type_from_value` — 从脚本值推导 ShaderType（第499-529行）

将一个 `ScriptValue` 转换为 `ShaderType`。优先级顺序：
1. `value_to_exact_type` 检查标量类型（f32、i32、u32、bool 等）
2. 颜色值 → `pod_vec4f`
3. `repr(u32)` 枚举变体 → `pod_u32`（检测 `_repr_u32_enum_value` 字段）
4. `pod_type` → `PodType`
5. `as_pod` → `Pod`
6. `as_pod_type` → `Pod`
7. 以上都不匹配 → `ShaderType::None`

### `find_highest_shader_io` — 在 prototype 链中查找最高层 ShaderIO 定义（第535-562行）

从 `io_self` 开始向 prototype 链上方遍历，对每个对象检查其 map 中的字段是否标记为 `shader_io`。返回找到的**最高层**（最祖先的）定义。这样如果子类覆盖了一个字段值（如 `x: #ffff`）但父类定义了 `x: shader.uniform(vec4f)`，编译器仍然使用父类的 uniform 类型。

### `get_io_self_field_value` — 获取 IoSelf 字段值（第567-581行）

优先调用 `find_highest_shader_io` 获取 shader IO 标记。如果找到了，返回 IO 标记对象和类型。如果没有，回退到通过 `heap.value` 获取普通的字段值。

### `shader_io_kind_matches` — 检查 IO kind 是否匹配（第583-602行）

两个 `ShaderIoKind` 是否逻辑兼容（如 `RustInstance` 和 `DynInstance` 共享相同的存储和前缀）。

### `shader_io_type_from_kind` — ShaderIoKind 转 ShaderIoType（第604-632行）

将 IO kind 转换为运行时 IO 类型 ID（如 `SHADER_IO_UNIFORM_BUFFER`、`SHADER_IO_VARYING` 等）。

### `infer_shader_io_type_from_pod_ty` — 从 Pod 类型推断 ShaderIO 类型（第634-643行）

隐式推断规则：`F32`/`F16`/`U32`/`I32` 标量→ `DYN_INSTANCE`，`Vec`/`Mat` 向量/矩阵→ `DYN_UNIFORM`。

### `infer_unmarked_shader_io_type` — 推断未标记字段的 ShaderIO 类型（第645-655行）

通过 `type_from_value` 获取值的 Pod 类型，如果匹配隐式规则则返回推断的 IO 类型。

### `handle_field` — 字段访问（第657-1366行）

这是整个模块最核心的方法。从栈上弹出字段 ID 和实例表达式，按实例类型分派：

**1. `ShaderType::Pod`**（第662-787行）：在 Pod 类型中查找字段类型。特殊处理 GLSL 和 WGSL 后端的 `unibuf_draw_pass` 实例——在 GLSL 中多视图相关字段（`camera_projection`、`camera_view` 等）加 `[int(VIEW_ID)]` 数组索引，在 WGSL 中则通过辅助函数访问。对普通字段生成 `instance.field_name`。

**2. `ShaderType::PodPtr`**（第788-822行）：指针类型，使用 `->` 语法：`ptr->field_name`。Metal 的 uniform buffer 使用此类型。

**3. `ShaderType::Texture`**（第823-830行）：纹理类型的字段访问被推迟为方法调用处理。将纹理表达式和字段 ID 推回栈上等待 `METHOD_CALL_ARGS` 处理。

**4. `ShaderType::ScopeObject`**（第831-1040行）：范围对象的字段访问。流程：
   - 在对象上查找字段值
   - 如果是函数对象 → 推回栈上作为方法调用接收者
   - 如果是 `repr(u32)` 枚举变体 → 直接输出 u32 常量
   - 如果是子对象（scope object 嵌套）→ 作为新的 `ScopeObject` 返回
   - 如果是普通属性 → 通过 `get_scope_value_pod_type` 获取 Pod 类型，注册为 `ScopeUniform` 到 `output.scope_uniforms` 和 `output.io`，访问时带 `ScopeUniform` 前缀
   - 如果值在 prototype 上找不到 → 尝试通过 `type_check` 结构获取字段类型（编译时类型信息），同样注册为 scope uniform

**5. `ShaderType::ScopeUniformBuffer`**（第1041-1114行）：范围 uniform 缓冲区的字段访问。注册缓冲区到 `output.scope_uniform_buffers`，生成 `buf_name.field_name` 或 `buf_name->field_name`（Metal 指针语法）。

**6. `ShaderType::IoSelf`**（第1115-1350行）：Shader IO 字段访问，最复杂的路径：
   - 通过 `get_io_self_field_value` 获取 shader IO 标记和类型
   - 如果找到显式标记：根据 IO kind（Uniform、Varying、Texture、VertexBuffer 等）生成带正确前缀的访问表达式。对 Texture 类型不推 Pod 类型而是推 `ShaderType::Texture`。对 UniformBuffer 在 Metal 中使用 `PodPtr`。
   - 如果无显式标记但符合隐式推断：使用推断规则（标量→instance，向量/矩阵→uniform）
   - 如果没有标记也不符合推断：在 `output.io` 中查找 `RustInstance` 字段（由 `pre_collect_rust_instance_io` 预收集）
   - 全部失败：报告错误，要求添加显式 IO 标记

### `handle_let_dyn` — let 变量声明（第1368-1435行）

弹出初始值表达式和变量 ID，决议具体类型。在 `shader_scope` 中通过 `define_let` 注册变量（不可变）。根据不同后端生成声明代码：
- **WGSL**：复合类型（向量、矩阵、结构体、枚举、数组）使用 `var` 声明（因为 WGSL 的 let 是不可变引用，不能用于可变的复合类型），标量使用 `let`
- **Metal/HLSL/GLSL**：`TypeName var_name = value;\n`
- **Rust**：`let mut var_name: TypeName = value;\n`（Rust 后端所有 shader 局部变量都是潜在的 mutable）

### `handle_var_dyn` — var 可变变量声明（第1437-1487行）

与 `handle_let_dyn` 类似，但在 `shader_scope` 中使用 `define_var` 注册（可变）。WGSL 始终使用 `var`，其他后端生成不变。

---

## 测试

文件末尾包含 `infer_unmarked_shader_io_rules_match_requested_defaults` 测试（第1490-1541行），验证 `infer_shader_io_type_from_pod_ty` 的隐式推断规则是否符合预期：标量（F32/I32/U32）→ `DYN_INSTANCE`，向量/矩阵 → `DYN_UNIFORM`，Bool 和 Struct → `None`。
