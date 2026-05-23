# `action.rs` — 动作系统

动作系统是 Makepad 框架中**事件→动作**分发机制的核心基础设施。它提供了一套类型擦除的、基于 `TypeId` 的多态动作容器，允许在 UI 树中安全地传递任意类型的动作数据，并通过类型安全的向下转型机制供具体组件消费。

---

## `ActionTrait` — 动作特质

```rust
pub trait ActionTrait: 'static {
    fn debug_fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result;
    fn ref_cast_type_id(&self) -> TypeId { TypeId::of::<Self>() }
}
```

- 所有可分发动作的**基础 trait**，约束 `'static` 以确保动作类型不包含非静态引用（这对跨线程投递至关重要）。
- **`debug_fmt`**：提供格式化输出，用于调试和日志记录。
- **`ref_cast_type_id`**：返回该 trait 对象的真实 `TypeId`。这是一个关键设计：因为 `ActionTrait` 被用作 trait object（`dyn ActionTrait`），直接调用 `TypeId::of::<dyn ActionTrait>()` 得不到具体类型。此方法通过虚函数表调用，返回实现者的具体 `TypeId`。

### Blanket Implementation

```rust
impl<T: 'static + Debug + ?Sized> ActionTrait for T { ... }
```

- 自动为所有实现了 `Debug` 的 `'static` 类型实现 `ActionTrait`。
- `debug_fmt` 直接委托给 `Debug::fmt`。
- `?Sized` 约束确保 `dyn ActionTrait` 本身也实现了 `ActionTrait`（尽管通常不直接使用）。

---

## `dyn ActionTrait` 方法

```rust
impl dyn ActionTrait {
    pub fn is<T: ActionTrait + 'static>(&self) -> bool { ... }
    pub fn downcast_ref<T: ActionTrait + 'static>(&self) -> Option<&T> { ... }
    pub fn downcast_mut<T: ActionTrait + 'static>(&mut self) -> Option<&mut T> { ... }
}
```

- **`is::<T>()`**：通过比较 `TypeId::of::<T>()` 和 `self.ref_cast_type_id()` 判断具体类型是否匹配。这里不能使用 `Any` trait，因为 `ActionTrait` 不是 `Any` 的子 trait，而是自己维护了一套 TypeId 机制。
- **`downcast_ref` / `downcast_mut`**：在类型匹配后，使用裸指针进行 `unsafe` 强制转型。这种实现避免了为每个动作类型引入额外的 `Any` trait 约束。安全性保证来自前一步的 `TypeId` 比对。
- 这些方法构成了动作树的"路由"基础：UI 组件遍历 actions 列表，对每个 action 调用 `downcast_ref` 检查是否是它关心的类型。

---

## 类型别名

```rust
pub type ActionSend = Box<dyn ActionTrait + Send>;
pub type Action = Box<dyn ActionTrait>;
pub type ActionsBuf = Vec<Action>;
pub type Actions = [Action];
```

- **`ActionSend`**：`Send` 约束的变体，用于跨线程投递（如 `UiRunner` 内部）。
- **`Action`**：普通的堆分配动作，是 UI 树中的标准传递单位。
- **`ActionsBuf`**：可增长的动态动作集合。
- **`Actions`**：切片类型，用于在不转移所有权的情况下遍历动作列表。

---

## `ActionDefaultRef` Trait

```rust
pub trait ActionDefaultRef {
    fn default_ref() -> &'static Self;
}
```

- 提供对类型的 `&'static` 静态默认引用。
- 用于在类型不匹配时提供安全的"空"返回值，避免使用 `Option` 或 panic。
- 通常由诸如 `None` 变体的枚举实现，返回一个静态的 "无操作" 实例。

---

## `ActionCast<T>` Trait

```rust
pub trait ActionCast<T> {
    fn cast(&self) -> T;
}
```

- **通用实现**：`Box<dyn ActionTrait>` → `ActionCast<T>`（要求 `T: ActionTrait + Default + Clone`）。
- `cast()` 尝试将 action 向下转型为 `T`，成功则 `clone()` 返回，失败则返回 `T::default()`。
- 这种"静默降级"设计使得动作处理代码更简洁：在类型不匹配时自动获得默认值而非 crash。

---

## `ActionCastRef<T>` Trait

```rust
pub trait ActionCastRef<T> {
    fn cast_ref(&self) -> &T;
}
```

- **`Box<dyn ActionTrait>` 实现**：尝试向下转型，失败时返回 `T::default_ref()`（需要 `T: ActionDefaultRef`）。
- **`Option<Arc<dyn ActionTrait>>` 实现**：如果 `Option` 是 `Some` 且类型匹配，返回转型后的引用；否则返回 `T::default_ref()`。
- 这种设计允许用 `&T` 引用而非 `T` 值来消费动作数据，避免了可能的克隆开销。

---

## `Cx` 上的动作方法

```rust
impl Cx {
    pub fn handle_action_receiver(&mut self) { ... }
    pub fn post_action(action: impl ActionTrait + Send) { ... }
    pub fn action(&mut self, action: impl ActionTrait) { ... }
    pub fn extend_actions(&mut self, actions: ActionsBuf) { ... }
    pub fn map_actions<F, G, R>(&mut self, f: F, g: G) -> R { ... }
    pub fn mutate_actions<F, G, R>(&mut self, f: F, g: G) -> R { ... }
    pub fn capture_actions<F>(&mut self, f: F) -> ActionsBuf { ... }
}
```

### `ACTION_SENDER_GLOBAL`

```rust
pub(crate) static ACTION_SENDER_GLOBAL: Mutex<Option<Sender<ActionSend>>> = Mutex::new(None);
```

- 全局静态的单向通道发送端。当第一个 `Cx` 实例创建时，会在新线程中创建通道，并将 `Sender` 注册到此全局变量。
- 通过 `Mutex<Option<...>>` 保护，因为 `post_action` 可能从任意线程调用。

### `Cx::handle_action_receiver()`
- 在事件循环中周期调用，**消费**全局通道中的异步动作。
- 使用 `try_recv()` 非阻塞地接收所有待处理的动作，追加到 `self.new_actions` 队列。
- 然后调用 `self.handle_actions()` 将 `new_actions` 分发到应用和组件的 `handle_event`。

### `Cx::post_action(action)`
- **静态方法**，可从任意线程调用。
- 加锁 `ACTION_SENDER_GLOBAL`，取出可选的 `Sender`。
- 通过 `sender.send(Box::new(action))` 发送动作。
- 若发送成功，调用 `SignalToUI::set_action_signal()` 通知 UI 线程有新动作待处理。
- **静默降级**：Mutx 中毒、Sender 不存在（Cx 未创建或已销毁）、channel 关闭（Cx 已 shutdown）时，动作被静默丢弃。这种设计对移动端后台/前台切换场景很重要——后台线程和 Cx 销毁之间存在竞态条件，不允许 panic。

### `Cx::action(action)`
- 在当前线程直接追加一个动作到 `new_actions` 队列。用于在同一线程（UI 线程）内合成动作。

### `Cx::extend_actions(actions)`
- 将给定的 `ActionsBuf` 追加到 `new_actions` 末尾。常用于 `capture_actions` 之后将部分或全部动作放回队列。

### `Cx::map_actions(f, g)`
- 高阶函数模式。执行闭包 `f`，捕获期间产生的所有动作，然后调用 `g` 对捕获的动作做转换，最后将转换后的动作放回。
- 实现方式：记录 `f` 执行前后的 `new_actions.len()`，取出中间产生的动作切片，调用 `g` 转换，再 `extend` 回去。

### `Cx::mutate_actions(f, g)`
- 类似于 `map_actions`，但 `g` 接收一个 `&mut [Action]` 切片进行原地修改，而非返回新的 `ActionsBuf`。

### `Cx::capture_actions(f)`
- 截获闭包 `f` 执行期间产生的所有动作，返回它们作为 `ActionsBuf`。
- 实现方式：将 `self.new_actions` 与一个空 Vec swap 出去，执行 `f`，产生的新动作进入 `new_actions`，然后将 `new_actions` 再次与之前的空 Vec swap，使得 `f` 期间的动作被"截获"到 `actions` 中。
- 这种"双 swap"技术避免了额外的内存分配。
- 如果 *不* 想将这些动作继续传播到 UI 树的其余部分，只需不调用 `extend_actions` 即可；如果想传播，调用 `extend_actions` 放回。

---

## 动作生命周期流程图

```
                    post_action (其他线程)
                          │
                          ▼
                  ACTION_SENDER_GLOBAL
                  (跨线程通道)
                          │
                          ▼
              Cx::handle_action_receiver()
              (事件循环中消费)
                          │
                          ▼
                  new_actions Vec
                          │
                          ▼
                  handle_actions()
                          │
                          ▼
              App::handle_event(cx, event)
                          │
                          ▼
              Event::Actions(actions) 遍历
                          │
                          ▼
              .downcast_ref::<T>() 类型匹配
                          │
                          ▼
                  消费动作数据
```

---

## 设计要点总结

1. **无侵入性**：任何 `'static + Debug` 类型都可以成为动作，无需实现特殊接口（除了 `ActionTrait` 的 blanket impl）。
2. **类型安全**：通过 `TypeId` 比对和 unsafe 转型实现，在性能和安全之间取得平衡。
3. **线程安全**：`ActionSend` 要求 `Send` 约束，通过全局通道跨线程传递。
4. **静默容错**：类型不匹配时返回 default/null 而非 panic；通道关闭时静默丢弃。
5. **灵活的动作处理**：`capture_actions` / `map_actions` / `mutate_actions` 提供了不同粒度的动作拦截/转换能力。
