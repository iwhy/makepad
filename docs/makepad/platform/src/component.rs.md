# `component.rs` — 组件注册表

`component.rs` 定义了 Makepad 框架的**组件注册表**系统。它提供了一种基于 `TypeId` 的运行时类型映射机制，允许在脚本运行时中按类型查找组件信息，支持跨模块的组件发现和动态类型转换。

---

## `ComponentInfo`

```rust
#[derive(Clone)]
pub struct ComponentInfo {
    pub name: LiveId,
}
```

- 组件信息的简单封装，目前仅包含一个 `LiveId` 作为组件名称。
- `Clone` 允许在需要时复制传递。
- `LiveId` 是 Makepad 脚本运行时的标识符类型（基于 `u64`），用于高效的编译期/运行期标识符匹配。

---

## `ComponentRegistry` Trait

```rust
pub trait ComponentRegistry {
    fn ref_cast_type_id(&self) -> TypeId;
    fn get_component_info(&self, name: LiveId) -> Option<ComponentInfo>;
    fn component_type(&self) -> LiveId;
}
```

- 所有组件注册器必须实现的 trait。
- **`ref_cast_type_id`**：返回实现者的具体 `TypeId`。这是类型安全向下转型的关键——与 `dyn ActionTrait` 中的相同设计模式。
- **`get_component_info(name)`**：根据给定的 `LiveId` 名称查询组件信息。返回 `Option`，允许未找到时返回 `None`。
- **`component_type`**：返回该注册器管理的组件类型的 `LiveId`。用于在注册表中快速匹配组件类型。

### `dyn ComponentRegistry` 方法

```rust
impl dyn ComponentRegistry {
    pub fn is<T: ComponentRegistry + 'static>(&self) -> bool { ... }
    pub fn downcast_ref<T: ComponentRegistry + 'static>(&self) -> Option<&T> { ... }
    pub fn downcast_mut<T: ComponentRegistry + 'static>(&mut self) -> Option<&mut T> { ... }
}
```

- 与 `action.rs` 中的 `dyn ActionTrait` 模式完全一致。
- **`is::<T>()`**：通过 `TypeId` 比对判断具体类型。
- **`downcast_ref` / `downcast_mut`**：类型匹配后使用裸指针 unsafe 转型。
- 这种设计使得 `ComponentRegistries` 可以存储 `Box<dyn ComponentRegistry>`，在需要时安全地转型回具体类型。

---

## `ComponentRegistries`

```rust
#[derive(Default, Clone)]
pub struct ComponentRegistries(pub Rc<RefCell<HashMap<TypeId, Box<dyn ComponentRegistry>>>>);
```

- 实际存储注册器的容器，使用 `Rc<RefCell<HashMap<TypeId, Box<dyn ComponentRegistry>>>>` 实现。
- **`Rc`**：引用计数，允许在多个地方共享同一个注册表。注册表通常在整个应用生命周期内单例存在。
- **`RefCell`**：内部可变性，允许在不可变引用（`&self`）下添加/修改注册器。
- **`HashMap<TypeId, ...>`**：以 `TypeId` 为键，保证每个具体类型只注册一次。
- **`Box<dyn ComponentRegistry>`**：类型擦除的注册器对象。

### 设计意图

`ComponentRegistries` 是组件系统的**全局名称服务**。当脚本运行时需要根据组件名称查找某个组件的类型信息时（例如在 Live 设计编辑器中解析组件引用），会遍历注册表中的所有注册器，调用 `component_type()` 匹配 `LiveId`，然后通过 `get_component_info()` 获取具体信息。

### `ComponentRegistries::new()`
- 创建一个空的注册表，初始化 `Rc<RefCell<HashMap>>`。

### `ComponentRegistries::find_component(ty, name)`

```rust
pub fn find_component(&self, ty: LiveId, name: LiveId) -> Option<ComponentInfo>
```

- 遍历注册表中所有条目。
- 对每个条目，先调用 `component_type()` 检查是否匹配给定的类型 ID `ty`。
- 若匹配，再调用 `get_component_info(name)` 查询组件信息。
- 一旦找到即返回，不继续遍历。
- 如果没有任何注册器匹配类型 ID，返回 `None`。

### `ComponentRegistries::get<T>()`

```rust
pub fn get<T: 'static + ComponentRegistry>(&self) -> std::cell::Ref<'_, T>
```

- 通过类型参数 `T` 直接在 HashMap 中查找注册器。
- 内部过程：`TypeId::of::<T>()` → HashMap 查询 → `downcast_ref::<T>()` → `Ref::map` 转换引用类型。
- 如果类型未注册，会 unwrap panic。
- 返回 `Ref<'_, T>`（RefCell 的 guard），确保借用规则在运行时被遵守。

### `ComponentRegistries::get_or_create<T>()`

```rust
pub fn get_or_create<T: 'static + Default + ComponentRegistry>(&self) -> std::cell::RefMut<'_, T>
```

- 类似 `get`，但如果指定类型尚未注册，**自动创建一个默认实例**并插入 HashMap。
- 使用 `HashMap::entry` API 避免重复查找：检查 Entry::Occupied 或 Entry::Vacant 分别处理。
- 返回 `RefMut<'_, T>`（RefCell 的可变借用 guard），允许调用者修改注册器内容。
- 此方法适合懒初始化场景：第一次需要时自动注册，后续直接获取。

---

## 与 `action.rs` 的对比

| 特性 | `action.rs` | `component.rs` |
|------|------------|---------------|
| 核心概念 | 动作分发 | 组件注册 |
| 类型擦除 | `Box<dyn ActionTrait>` | `Box<dyn ComponentRegistry>` |
| 类型标识 | `ref_cast_type_id()` | `ref_cast_type_id()` |
| 向下转型 | `downcast_ref` / `downcast_mut` | `downcast_ref` / `downcast_mut` |
| 容器 | `Vec<Action>` | `HashMap<TypeId, Box<dyn ...>>` |
| 查询方式 | 线性遍历 + 类型匹配 | HashMap 直接索引或线性遍历 |
| 创建策略 | 直接构造 | `get_or_create` 懒创建 |

---

## 使用流程示例

```
                    Studio/编辑器
                         │
                         ▼  "查找类型为 Button 的组件"
              ComponentRegistries::find_component(
                  component_type: "Button" 的 LiveId,
                  name: "my_button" 的 LiveId
              )
                         │
                         ▼
              遍历 HashMap 所有值
                         │
                         ▼
              entry.component_type() == ty ?
                         │
                    ┌────┴────┐
                  是│         │否
                    ▼         │
              get_component_ │  继续下一个
              info(name)     │
                    │        │
                    ▼        ▼
              Some(info)   continue
                         │
                         ▼
                  返回 ComponentInfo

// 代码中直接按类型获取
let reg = registries.get::<ButtonRegistry>();
reg.get_component_info(id!(primary_button));
```
