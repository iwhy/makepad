# derive_widget.rs — Widget 家族派生宏实现

## 概述

`derive_widget.rs` 实现了五个派生宏，分别为 `#[derive(Widget)]`（主宏，内部聚合四个子生成器）、`WidgetNode`、`WidgetRegister`、`WidgetRef` 和 `WidgetSet`。它们是 Makepad UI 框架的核心基础设施，为自定义 widget 提供类型安全的访问接口和 trait 实现。

## 主入口: `derive_widget_impl`

```rust
#[proc_macro_derive(Widget, attributes(walk, deref, redraw, find, wrap, area, event, visible, action_data, uid, cast))]
```

该主宏调用四个子生成器并将输出拼接：
1. `derive_widget_node_impl` — 生成 `WidgetNode` trait 实现
2. `derive_widget_register_impl` — 生成 `WidgetRegister` trait 实现
3. `derive_widget_ref_impl` — 生成 `StructNameRef` 新类型 + 访问 trait
4. `derive_widget_set_impl` — 生成 `StructNameSet` 新类型 + 批量访问 trait

---

## 1. `derive_widget_node_impl` — WidgetNode trait

### 支持属性

| 属性 | 说明 | 多个？ | 描述 |
|------|------|--------|------|
| `#[walk]` | Walk 字段 | 单 | widget 的布局行走策略 |
| `#[deref]` | 委托字段 | 单 | 将大部分方法委托给内部 widget |
| `#[redraw]` | 重绘字段 | 多 | 调用 `.redraw(cx)` 的字段列表 |
| `#[find]` | 子 widget 搜索 | 多 | 用于 `children()` 和 `find_widgets_from_point()` 的字段 |
| `#[wrap]` | 包装字段 | 单 | 完全委托给内部 View/widget 的所有方法 |
| `#[area]` | Area 字段 | 单 | 返回 widget 的命中检测区域 |
| `#[visible]` | 可见性字段 | 单 | `bool` 类型字段，用于可见性控制 |
| `#[action_data]` | 动作数据 | 单 | 用于存储 `Arc<dyn ActionTrait>` |
| `#[uid]` | 直接 UID | 单 | 直接返回 `WidgetUid` 的字段（不与 `#[deref]` 共存） |
| `#[cast]` | 类型转换 | 多 | 为 `cast_inner_any(_mut)` 注册内部类型 |

### 结构体设计约束

- **必须有且仅有一个**：`#[uid]` 字段、`#[deref]` 字段、或 `#[wrap]` 字段
- **不允许同时**有 `#[uid]` 和 `#[deref]`
- **必须有 `#[redraw]` 或 `#[deref]` 或 `#[wrap]`** 来提供 `redraw()` 方法
- 顶层 `#[designable]` 属性（通过 main_attribs 获取）控制是否生成 `widget_design()` 方法

### 生成的 WidgetNode 方法

#### walk 方法
- 有 `#[walk]`：返回 `self.walk_field` 的值（类型为 `Walk`）
- 有 `#[deref]`：委托给 `self.deref_field.walk(cx)`
- 默认：返回 `Walk::default()`

#### redraw 方法
- 有 `#[redraw]`：对所有标记字段依次调用 `field.redraw(cx)`
- 有 `#[deref]`：委托给 `self.deref_field.redraw(cx)`
- 无则：**编译错误**

#### visible / set_visible
- 有 `#[wrap]`：完全委托给 wrap 字段
- 有 `#[visible]`：直接读写 bool 字段，`set_visible` 自动调用 `self.redraw(cx)`
- 有 `#[deref]`：委托给 deref 字段
- 否则：不生成

#### area
- 有 `#[wrap]`：委托给 wrap 字段
- 有 `#[area]`：返回 `.area()` 字段
- 有 `#[deref]`：委托给 deref 字段
- 有 `#[redraw]`：返回第一个 redraw 字段的 area

#### children / find_widgets_from_point
- 有 `#[wrap]`：委托给 wrap 字段
- 有 `#[find]`：对所有 find 字段依次调用
- 有 `#[deref]`：委托给 deref 字段
- 无则：不生成

#### skip_widget_tree_search
- 有 `#[wrap]` 或 `#[deref]`：委托给对应字段

#### Selection API
- 有 `#[wrap]`：完全委托给 wrap 字段
- 有 `#[find]`：遍历 find 字段返回第一个非零/非空结果
- 有 `#[deref]`：委托给 deref 字段
- 方法：`selection_text_len`、`selection_point_to_char_index`、`selection_set`、`selection_clear`、`selection_select_all`、`selection_get_text_for_range`、`selection_get_full_text`

#### set_action_data / action_data
- 有 `#[action_data]`：生成 `set_action_data` 和 `action_data` 方法

#### cast_inner_any / cast_inner_any_mut
- 有 `#[cast]`：为每个 cast 字段生成 TypeId 匹配分支，可同时注册多个内部类型

#### widget_uid
- 有 `#[deref]`：委托给 deref 字段
- 有 `#[uid]`：直接返回字段值
- 有 `#[wrap]`：委托给 wrap 字段

#### widget_design
- 顶层属性包含 `designable` 时生成：`fn widget_design(&mut self) -> Option<&mut dyn WidgetDesign> { Some(self) }`

---

## 2. `derive_widget_register_impl` — WidgetRegister trait

### 生成的代码

```rust
impl WidgetRegister for StructName {
    fn register_widget(vm: &mut ScriptVm) -> ScriptValue {
        register_widget!(vm.cx_mut(), StructName);
        Self::script_component(vm)
    }
}
```

该实现调用 `register_widget!` 宏将 widget 类型注册到运行时类型系统，然后调用 `script_component` 返回脚本值。这是 widget 在脚本层可用的前提。

---

## 3. `derive_widget_ref_impl` — WidgetRef 新类型 + 访问 trait

### 生成的内容

对于结构体 `MyWidget`，生成：

#### MyWidgetRef 新类型
```rust
#[derive(Clone, Debug)]
pub struct MyWidgetRef(WidgetRef);

impl Deref for MyWidgetRef { type Target = WidgetRef; ... }
impl DerefMut for MyWidgetRef { ... }
```

#### MyWidgetRef 方法
- `has_widget(widget) -> MyWidgetRef` — 检查内部 WidgetRef 是否匹配，匹配返回自身副本，不匹配返回默认（空）
- `borrow() -> Option<Ref<'_, MyWidget>>` — 借用内部 MyWidget
- `borrow_mut() -> Option<RefMut<'_, MyWidget>>` — 可变借用
- `borrow_if_eq(widget) -> Option<Ref<'_, MyWidget>>` — 条件借用（仅当 WidgetRef 相等时）
- `borrow_mut_if_eq(widget) -> Option<RefMut<'_, MyWidget>>` — 条件可变借用

#### MyWidgetWidgetRefExt trait (for WidgetRef)
```rust
pub trait MyWidgetWidgetRefExt {
    fn my_widget(&self, cx: &Cx, path: &[LiveId]) -> MyWidgetRef;
    fn as_my_widget(&self) -> MyWidgetRef;
}
```

`my_widget(path)` 通过路径查找 widget，`as_my_widget` 直接包装当前引用。

#### MyWidgetWidgetExt trait (for any Widget)
```rust
pub trait MyWidgetWidgetExt {
    fn my_widget(&self, cx: &Cx, path: &[LiveId]) -> MyWidgetRef;
}
impl<T: Widget> MyWidgetWidgetExt for T { ... }
```

#### 工具函数
- `camel_case_to_snake_case("MyWidget")` → `"my_widget"` — 用于生成 getter/setter 的方法名

---

## 4. `derive_widget_set_impl` — WidgetSet 新类型 + 批量访问 trait

### 生成的内容

对于结构体 `MyWidget`，生成：

#### MyWidgetSet 新类型
```rust
#[derive(Clone, Debug)]
pub struct MyWidgetSet(WidgetSet);
impl Deref for MyWidgetSet { type Target = WidgetSet; ... }
impl DerefMut for MyWidgetSet { ... }
```

#### MyWidgetSet 方法
- `has_widget(widget) -> MyWidgetRef` — 检查集合中是否包含指定 widget
- `iter() -> MyWidgetSetIterator` — 迭代器返回

#### MyWidgetSetWidgetSetExt trait (for WidgetSet)
```rust
pub trait MyWidgetSetWidgetSetExt {
    fn my_widget_set(&self, cx: &Cx, paths: &[&[LiveId]]) -> MyWidgetSet;
    fn as_my_widget_set(&self) -> MyWidgetSet;
}
```

#### MyWidgetSetWidgetRefExt trait (for WidgetRef)
```rust
impl MyWidgetSetWidgetRefExt for WidgetRef {
    fn my_widget_set(&self, cx: &Cx, paths: &[&[LiveId]]) -> MyWidgetSet { ... }
}
```

#### MyWidgetSetWidgetExt trait (for any Widget)
```rust
impl<T: Widget> MyWidgetSetWidgetExt for T {
    fn my_widget_set(&self, cx: &Cx, paths: &[&[LiveId]]) -> MyWidgetSet { ... }
}
```

#### MyWidgetSetIterator
```rust
pub struct MyWidgetSetIterator<'a> { iter: WidgetSetIterator<'a> }
impl Iterator for MyWidgetSetIterator<'_> { type Item = MyWidgetRef; ... }
```

## camel_case_to_snake_case 工具函数

将驼峰命名转换为蛇形命名：
- 连续大写字母被视为一个词（如 `URLParser` → `url_parser`）
- 首字母大写后的首个大写字母前插入下划线
- 示例：`MyWidget` → `my_widget`，`StudioCodeEditor` → `studio_code_editor`

## 属性组合模式总结

```
               ┌──────────────────────────────────────────────┐
               │          派生宏 (#[derive(Widget)])          │
               ├──────────┬──────────┬──────────┬────────────┤
               │WidgetNode│Register  │ WidgetRef │ WidgetSet │
               │  trait   │  trait   │ Ref + Ext │ Set + Ext │
               └──────────┴──────────┴──────────┴────────────┘
                    ▲
                    │ 属性标记决定行为
        ┌───────┬───┼───┬───────┬───────┬───────┐
        │deref  │find│walk│redraw │visible│ uid   │ ...
        │委托给  │子  │布局 │重绘   │可见性 │UID    │
        │内部wg  │搜索│策略 │触发器 │控制   │直接   │
```

## 旧代码注释

文件末尾包含已注释的旧版 `derive_widget_impl` 实现，以及部分已注释的 `Option<WidgetRef>` trait 实现。这些代码不再使用，保留作为参考。
