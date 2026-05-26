# `traits.rs` 源码解读

**路径:** `platform/script/src/traits.rs`
**行数:** 593
**核心职责:** 定义 Makepad 脚本引擎的核心 trait 体系，是所有 Rust 类型与脚本运行时互操作的接口层。包括 `ScriptHook`（生命周期钩子）、`ScriptNew`（创建与类型注册）、`ScriptApply`（值应用与序列化）等核心接口。

---

## `ScriptDeriveMarker`

**路径:** 第11行

空 trait，作为 proc-macro 生成的标记，用于泛型约束中区分"已被 derive 宏处理过的类型"。

---

## `ScriptTypeId`

**路径:** 第13行

类型别名，等于 `std::any::TypeId`。每种 Rust 类型有唯一的 TypeId，脚本引擎用它做类型匹配。

---

## Trait `ScriptHook`

**路径:** 第16-108行

脚本对象的生命周期钩子接口。在 apply 流程中由 `ScriptVm` 自动调用。

### 核心钩子

**`on_before_apply` / `on_after_apply`**（第18-54行）
在 apply 前后调用的钩子，默认空实现。派生宏可以覆盖。

**`on_before_dispatch`**（第27-45行）
在 apply 分发前调用。默认根据 Apply 类型转发到：
- `Apply::New` → `on_before_new_scoped`
- `Apply::Reload` / `Apply::ScriptReapply` → `on_before_reload_scoped`

**`on_after_dispatch`**（第56-69行）
在 apply 分发后调用。转发到 `on_after_new_scoped` / `on_after_reload_scoped`，然后总是调用 `on_alive()`。

**`on_custom_apply`**（第71-79行）
返回 true 时跳过自动生成的 apply 代码，使用自定义 apply 逻辑。

**`on_type_check`**（第82-84行）
静态方法，用于 proc-macro 反射生成的类型检查逻辑。

### 别名方法

**`on_before_new` / `on_before_reload` / `on_after_new` / `on_after_reload`**（第90-93行）
简洁版本，不带 Scope 参数。

**`on_before_new_scoped` / `on_before_reload_scoped` / `on_after_new_scoped` / `on_after_reload_scoped`**（第96-107行）
带 Scope 参数的版本，默认委托到无 Scope 版本。

---

## Trait `ScriptHookDeref`

**路径:** 第110-127行

为 `#[deref]` 字段设计的钩子扩展。在 apply 流程中，如果目标类型通过 `Deref` 委托给内部字段，可在 apply 前后通过此 trait 插入自定义逻辑。

---

## 结构体 `ScriptTypeProp` / `ScriptTypeProps`

**路径:** 第129-180行

用于存储类型反射信息的数据结构。

### `ScriptTypeProp`（第129-133行）
- `order: u32`：字段顺序（排序用）
- `ty: ScriptTypeId`：字段类型

### `ScriptTypeProps`（第135-180行）
- `props: LiveIdMap<LiveId, ScriptTypeProp>`：字段名到属性的映射
- `rust_instance_start: u32`：标记 Rust 实例字段在 props 列表中的起始索引

#### 方法
- **`insert(id, ty)`**：按当前长度作为 order 插入新属性
- **`mark_rust_instance_start()`**：标记实例字段起始位置（在 `#[deref]` 字段处理前调用）
- **`iter_ordered()`**：按 order 排序返回所有 (LiveId, TypeId) 对
- **`iter_rust_instance_ordered()`**：仅返回 order ≥ rust_instance_start 的字段（跳过配置字段），按 order 排序。着色器编译器使用此方法来构建 `RustInstance` 结构体布局

---

## 结构体 `ScriptTypeObject`

**路径:** 第182-187行

已注册脚本类型的描述：
- `type_id: ScriptTypeId`：唯一类型标识
- `check: fn(...)`：类型检查函数指针（避免闭包的堆分配）
- `proto: ScriptValue`：该类型的原型对象
- `name: Option<LiveId>`：可选的类型名（用于错误消息）

---

## 结构体 `ScriptTypeCheck`

**路径:** 第189-194行

脚本类型的完整类型描述：
- `props: ScriptTypeProps`：属性列表
- `object: Option<ScriptTypeObject>`：类型对象（pod 类型没有）
- `is_repr_u32_enum: bool`：是否是 `repr(u32)` 枚举

---

## 函数 `register_type_inner`

**路径:** 第200-225行

非泛型辅助函数，减少单态化膨胀。将 type_id、proto、props、check 函数名等组合成 `ScriptTypeCheck` 并注册到 heap。若 proto 是对象则设置类型标签。

---

## Trait `ScriptNew`

**路径:** 第228-542行

创建和注册脚本类型的核心接口。

### 静态方法

**`script_type_name()`**（第234-236行）
返回类型的可读名称（默认 None，derive 宏覆盖）。

**`is_repr_u32_enum()`**（第240-243行）
是否 `repr(u32)` 枚举。

**`script_type_check(heap, value)`**（第245-254行）
类型检查：先调用 `on_type_check`，再检查 value 是否是 object 且 type_id 匹配。

**`script_pod(vm)`**（第260-402行）
构造该类型的 Pod 结构体定义（用于着色器兼容性）：
1. 递归计算字段的 `rust_repr_layout_for_type_id`（获取 size 和 align）
2. 只包含 `iter_rust_instance_ordered` 返回的实例字段（跳过配置字段）
3. 对每个字段递归查找 PodType
4. 严格对齐验证：断言 Rust `repr(C)` 偏移与 shader Pod 偏移一致
5. 最终断言 Rust 总大小与 Pod 总大小一致（防止 GPU 数据错位）

**`script_default(vm)`**（第404-410行）
构建 proto 后创建默认实例并序列化。

**`script_reload_default(vm)`**（第412-422行）
从 heap 的类型默认缓存中获取 reload 使用的默认值。

**`from_script_mod(vm, f)`**（第441-454行）
执行 `script_mod!` 块并将结果值 apply 到当前类型实例。若返回 nil 则 panic 提示脚本块必须以表达式结尾。

### 实例方法

**`script_new(vm)`**（第427行）
纯虚方法：在 VM 上下文中创建该类型的新实例。

**`script_new_with_default(vm)`**（第429-439行）
优先查找 heap 中的类型默认值，若存在则 from_value 创建，否则调用 script_new。

### 原型构建

**`script_proto(vm)`**（第476-493行）
缓存的双检锁模式：若 heap 中已有注册类型则直接返回 proto；否则构造 `ScriptTypeProps`，调用 `proto_build`，并注册。

**`script_proto_build(vm, props)`**（第495-502行）
创建空对象，依次调用 `script_proto_props`（填充 props）、`on_proto_build`（钩子）、`on_proto_methods`（注册方法）。

### API/组件/着色器注册

- **`script_api(vm)`**：注册为 API 类型（freeze_api）
- **`script_component(vm)`**：注册为组件类型（freeze_component）
- **`script_shader(vm)`**：注册为着色器类型（freeze_shader）
- **`script_ext(vm)`**：注册为扩展类型（freeze_ext）

### `script_enum_lookup_variant(vm, variant)`（第531-541行）

在已注册类型中查找枚举变体：从 proto 对象中按 id 查找 value。

---

## Trait `ScriptApply`

**路径:** 第544-577行

值应用与序列化接口。

### 方法
- **`script_type_id()`**：返回当前实例的 TypeId。
- **`script_apply(vm, apply, scope, value)`**：将 ScriptValue 应用到当前实例（核心方法）。
- **`script_to_value(vm)`**：将当前实例序列化为 ScriptValue。
- **`script_to_value_props(vm, obj)`**：将属性写入指定对象（优化用）。
- **`script_source()`**：返回关联的脚本源对象。
- **`script_apply_eval(vm, script_mod)`**：执行 ScriptMod 代码并 apply 结果到自身。先将 `__script_source__` 设为当前对象的 source，再 eval，最后以 `Apply::Eval` 分发。

---

## Trait `ScriptApplyDefault`

**路径:** 第579-589行

可选默认值机制。若 `script_apply_default` 返回 `Some(value)`，则在 apply 流程中使用该值而非传入值。

---

## Trait `ScriptReset`

**路径:** 第591-592行

重置接口：将对象重置到指定值，用于 live reload 时的差异化重置逻辑。
