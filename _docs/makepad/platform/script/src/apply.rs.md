# `apply.rs` 源码解读

**路径:** `platform/script/src/apply.rs`
**行数:** 251
**核心职责:** 定义脚本引擎的值应用基础设施，包括类型擦除的作用域数据容器（`ScopeDataRef`/`ScopeDataMut`）、`Scope` 上下文和 `Apply` 枚举（标记 apply 操作的来源与语义）。

---

## `ScopeDataRef<'a>` / `ScopeDataMut<'a>`

**路径:** 第7-27行

类型擦除的作用域数据容器，用于在 apply 流程中传递额外的上下文数据，避免泛型参数。

### `ScopeDataRef`（第7-8行）
不可变引用包装：`Option<&'a dyn Any>`。

- **`get<T>()`**（第14-16行）：尝试将内部引用向下转换为 `&T`。

### `ScopeDataMut`（第10-11行）
可变引用包装：`Option<&'a mut dyn Any>`。

- **`get<T>()`**（第20-22行）：不可变地获取 `&T`。
- **`get_mut<T>()`**（第24-26行）：可变地获取 `&mut T`。

---

## `Scope<'a, 'b>`

**路径:** 第33-121行

apply 操作的上下文对象，包含两个类型擦除的数据容器和一个索引。

### 字段
- `data: ScopeDataMut<'a>`：可变数据（传递给 apply 的上下文）
- `props: ScopeDataRef<'b>`：不可变属性数据
- `index: usize`：索引计数器（用于 PortalList 等需要索引的 apply）

### 构造方法

| 方法 | data | props | index |
|------|------|-------|-------|
| `with_data(v)` | Some(v) | None | 0 |
| `with_data_props(v, w)` | Some(v) | Some(w) | 0 |
| `with_props(w)` | None | Some(w) | 0 |
| `with_data_index(v, index)` | Some(v) | None | index |
| `with_data_props_index(v, w, index)` | Some(v) | Some(w) | index |
| `with_props_index(w, index)` | None | Some(w) | index |
| `empty()` | None | None | 0 |

### `override_props(props, f)`（第97-106行）

临时替换 props 后执行闭包，闭包完成后恢复原始 props。

### `override_props_index(props, index, f)`（第108-120行）

临时替换 props 和 index 后执行闭包，闭包完成后恢复原始值。

---

## `Apply`

**路径:** 第127-251行

标记 apply 操作来源的枚举。

### 变体

| 变体 | 语义 |
|------|------|
| `New` | 初始创建/绑定 |
| `Reload` | LiveEdit 热重载：DSL 本身发生了变化，模板值是新的真理来源 |
| `ScriptReapply` | 堆变更广播（`cx.request_script_reapply()`）：模板未变，共享堆对象引用需要刷新生效 |
| `Animate` | 动画驱动 |
| `Eval` | 运行时 eval 评估 |
| `Default(usize)` | 默认值设定（带索引） |

### 查询方法

- **`is_from_script()`**：New、Reload、ScriptReapply、Eval 返回 true。标记该操作是否源自脚本。
- **`is_template_apply()`**（第165-171行）：New 和 Reload 返回 true。标记是否需要更新 `#[source]` 字段。排除 Eval（临时对象会被 GC）和 ScriptReapply（模板未变）。
- **`is_new()`**：仅 New。
- **`is_reload()`**（第192-198行）：Reload 和 ScriptReapply 均返回 true。大部分 widget 的模板重应用逻辑应使用此宽泛语义。
- **`is_live_edit_reload()`**：仅 Reload。只在 DSL 源码真正变化时才应返回 true 的操作（如重新运行 script_mod 脚手架）。
- **`is_script_reapply()`**（第217-222行）：仅 ScriptReapply。不应被模板默认值覆盖的字段（如 `String`/`ArcStringMut`）应在此早期返回。
- **`is_animate()` / `is_eval()` / `is_default()`**：对应变体的判断。
- **`as_default()`**：返回 `Default(usize)` 中的 usize 值。
