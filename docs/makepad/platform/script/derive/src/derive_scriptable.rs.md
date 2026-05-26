# `derive_scriptable.rs` 源码解读

**路径:** `platform/script/derive/src/derive_scriptable.rs`
**行数:** 1042
**核心职责:** 实现 `#[derive(Script)]` 和 `#[derive(ScriptHook)]` 派生宏的代码生成引擎，为 struct 和 enum 自动生成 `ScriptNew`、`ScriptApply`、`ScriptHookDeref`、`ScriptDeriveMarker` 等全套 trait 实现

---

## 类型定义

### `struct EnumItem`（第 395 行，函数内部）
枚举变体的描述结构体，包含 `name`（变体名称）、`attributes`（属性列表）、`kind: EnumKind`（变体种类）、`discriminant: Option<TokenStream>`（`repr(u32)` 枚举的判别式表达式）。

### `enum EnumKind`（第 402 行，函数内部）
描述枚举变体种类的枚举，三个变体：
- `Bare` — 无字段的单元变体
- `Named(Vec<StructField>)` — 命名字段的结构体变体
- `Tuple(Vec<TokenStream>)` — 元组字段变体

### `impl EnumItem`（第 409 行，函数内部）
为 `EnumItem` 提供 `gen_new(&self, tb: &mut TokenBuilder) -> Result<(), TokenStream>` 方法，根据变体种类生成 `Self::VariantName` 构造代码：`Bare` 仅输出变体名，`Named` 输出 `{default_values}`，`Tuple` 输出 `(default_values)`。

---

## 顶层函数

### `pub fn derive_script_impl(input: TokenStream) -> TokenStream`
**第 5 行**

`#[derive(Script)]` 的入口函数。创建 `TokenParser` 和 `TokenBuilder`，调用 `derive_script_impl_inner` 生成代码。若内层函数返回错误则输出错误 token 流，否则输出构建的完整 token 流。

### `fn derive_script_impl_inner(parser, tb) -> Result<(), TokenStream>`
**第 15 行**

核心代码生成逻辑，分为 struct 和 enum 两个分支。

**struct 分支（第 21-385 行）：**

1. **Deref 实现（第 44-73 行）**：如果存在 `#[deref]` 字段，自动生成 `Deref` 和 `DerefMut` trait 实现，将解引用转发到该字段。这是实现 widget 继承的机制——子 widget 通过 `#[deref]` 指向父 widget 类型。
2. **ScriptDeriveMarker（第 76-81 行）**：为空 trait 生成标记实现，标识该类型已通过派生宏处理。
3. **ScriptHookDeref（第 83-99 行）**：生成 `ScriptHookDeref` 实现，将 `on_deref_before_apply` 和 `on_deref_after_apply` 转发到 `ScriptHook` trait 的四个方法(`on_before_apply`、`on_before_dispatch`、`on_after_apply`、`on_after_dispatch`)。
4. **ScriptApply（第 101-284 行）**：
   - `script_type_id`：返回 `ScriptTypeId::of::<Self>()`
   - `script_apply`：应用脚本值的核心实现。先检查 `on_custom_apply` 钩子，再调用 `on_deref_before_apply`。按属性类型处理字段：
     - `#[source]`：仅在非 Eval 且来自脚本时更新，防止临时原型对象永久化
     - `#[live]` / `#[apply_default]`：通过 `value_for_apply` 获取字段值，支持 reload 时的默认值回退
     - `#[splat]` / `#[walk]` / `#[layout]`：接收整个 value 进行递归应用
     - `#[deref]`：**在 `apply_default` 之前执行**，重要顺序保证——动画器的状态驱动值不会在重应用时被模板默认值覆盖
     - `#[apply_default]`：最后执行，调用 `script_apply_default` 返回的 apply block 递归应用到整个 widget，确保动画状态优先级最高
   - `script_to_value`：将当前 struct 序列化为脚本值，创建原型对象并设置所有属性
   - `script_to_value_props`：逐个序列化属性到已有对象上，处理 deref、walk/layout/splat、live/apply_default 三种属性的传播
   - `script_source`（条件生成）：如果存在 `#[source]` 字段，返回该字段的脚本对象引用

5. **ScriptNew（第 288-379 行）**：
   - `script_type_id_static` / `script_type_name`：返回类型 ID 和名称
   - `script_new`：按字段属性生成构造函数：
     - `#[new]` / `#[live]` / `#[apply_default]` / `#[deref]`：调用 `ScriptNew::script_new_with_default(vm)`
     - `#[uid]`：调用 `WidgetUid::new()`
     - `#[rust]` / `#[source]`（无参数）：调用 `Default::default()`
     - 属性带参数：调用 `(args).into()` 包装
   - `script_proto_props`：构建类型的原型属性系统。先处理 `#[deref]` 字段（标记 `rust_instance_start` 后递归父类属性），然后处理 `#[walk]`/`#[layout]`/`#[splat]`（级联属性但不标记起始），最后注册 `#[live]`/`#[apply_default]` 字段的类型 ID，构成运行时类型检查的基础。

**enum 分支（第 386-857 行）：**

1. 解析枚举变体（第 436-486 行）：遍历所有变体，按 `Bare`、`Tuple`、`Named` 分类，支持 `#[pick]`/`#[default]` 标记默认变体，支持 `= expr` 格式的判别式（用于 `repr(u32)` 枚举）。
2. **ScriptDeriveMarker（第 494-499 行）**：生成标记 trait 实现。
3. **ScriptNew（第 503-664 行）**：
   - `script_new`：构造 `#[pick]` 标记的默认变体
   - `script_default`：构建原型并调用默认值
   - `script_new_with_default`：始终返回 pick 变体
   - `script_reload_default`：检查类型默认值是否存在
   - `script_type_check`：运行时类型检查，匹配变体的 LiveId
   - `is_repr_u32_enum`（条件）：如果存在判别式返回 `true`
   - `script_proto_build`：构建枚举 API 对象。`Bare` 变体创建冻结原型；`Tuple` 变体注册为方法，带参数数量/类型校验；`Named` 变体创建带属性的原型，注册 `ScriptTypeCheck` 以便类型验证
4. **ScriptApply（第 668-852 行）**：
   - `script_apply`：根据值的 `root_proto` ID 匹配到对应变体。`Bare` 直接赋值；`Tuple`/`Named` 在变体匹配时保持已有值不变(仅首次创建)，然后递归应用子字段
   - `script_to_value`：序列化为脚本值，`Bare` 查找变体原型；`Tuple` 创建新原型并逐个 push 字段；`Named` 创建新原型并逐个 set 属性

### `pub fn derive_script_hook_impl(input: TokenStream) -> TokenStream`
**第 860 行**

`#[derive(ScriptHook)]` 的实现。解析 struct 或 enum 声明，生成空的 `impl ScriptHook for TypeName {}` 实现块。所有 `ScriptHook` 的默认方法（如 `on_before_apply`、`on_after_apply`、`on_custom_apply` 等）由 trait 自带默认实现。

---

## 重要设计决策

**字段应用顺序（第 182-223 行）：** `#[deref]` 字段的 script_apply 必须在 `#[apply_default]` 之前执行。这解决了动画器状态在 `Apply::ScriptReapply` 时被模板默认值覆盖的问题——先恢复模板默认（deref），再应用动画器的状态覆盖（apply_default），确保 widget 在整个 apply 过程中不会"闪回"到模板默认状态。

**属性宏列表（第 34-46 行）：** 每个字段必须标注属性，支持的属性包括：
- `#[live]` — 可脚本映射的字段
- `#[rust]` — 仅 Rust 运行时字段（不可从脚本访问）
- `#[apply_default]` — 具备默认值应用的脚本字段
- `#[deref]` — 基类/父类字段（自动实现 Deref/DerefMut）
- `#[source]` — 脚本对象源引用
- `#[new]` — 带默认值构造
- `#[walk]` / `#[layout]` / `#[splat]` — 布局属性字段（接收整个值）
- `#[uid]` — WidgetUid 字段
- `#[pick]` — 枚举默认变体标记
