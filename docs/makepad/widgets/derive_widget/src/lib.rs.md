# lib.rs — derive_widget 库入口

## 概述

`lib.rs` 是 `makepad-derive-widget` proc-macro 库的入口文件。它定义了五个公开的 proc-macro 派生宏，并将实现委托给 `derive_animator.rs` 和 `derive_widget.rs` 两个内部模块。

## 模块结构

```rust
mod derive_animator;   // Animator 派生宏实现
mod derive_widget;     // Widget/WidgetRef/WidgetRegister/WidgetSet 派生宏实现
```

## 导出的派生宏

### `#[derive(Widget)]`

```rust
#[proc_macro_derive(Widget, attributes(
    apply_default, walk, deref, redraw, find, wrap,
    area, event, visible, action_data, uid, cast,
))]
pub fn derive_widget(input: TokenStream) -> TokenStream
```

**这是最主要的派生宏**，一次性生成四个 trait 实现：
1. `WidgetNode` — widget 树节点接口（walk、redraw、children、area 等）
2. `WidgetRegister` — widget 脚本注册接口
3. `WidgetRef` — 类型安全的单一 widget 引用包装 + 访问 trait
4. `WidgetSet` — 类型安全的批量 widget 引用包装 + 访问 trait

**支持属性**：
| 属性 | 用途 |
|------|------|
| `apply_default` | 应用默认值初始化 |
| `walk` | 标记 Walk 布局字段 |
| `deref` | 标记委托字段 |
| `redraw` | 标记重绘字段（可多个） |
| `find` | 标记子 widget 搜索字段（可多个） |
| `wrap` | 标记完全包装字段 |
| `area` | 标记 Area 字段 |
| `event` | 事件相关（在推导过程中未直接使用） |
| `visible` | 标记可见性 bool 字段 |
| `action_data` | 标记动作数据字段 |
| `uid` | 标记 WidgetUid 直接字段 |
| `cast` | 标记类型转换字段（可多个） |

### `#[derive(Animator)]`

```rust
#[proc_macro_derive(Animator, attributes(animator,))]
pub fn derive_animator(input: TokenStream) -> TokenStream
```

为带有 `animator: Animator` 字段的结构体生成 `AnimatorImpl` trait 实现。支持属性 `#[animator]`（标记在字段上）。

### `#[derive(WidgetRef)]`

```rust
#[proc_macro_derive(WidgetRef)]
pub fn derive_widget_ref(input: TokenStream) -> TokenStream
```

单独生成 `StructNameRef` 新类型 + 相关的 `WidgetRefExt` 和 `WidgetExt` trait。不包含 `WidgetNode` 或 `WidgetRegister` 的实现。

### `#[derive(WidgetRegister)]`

```rust
#[proc_macro_derive(WidgetRegister)]
pub fn derive_widget_register(input: TokenStream) -> TokenStream
```

单独生成 `WidgetRegister` trait 实现（`register_widget` 方法），不包含其他生成内容。

### `#[derive(WidgetSet)]`

```rust
#[proc_macro_derive(WidgetSet)]
pub fn derive_widget_set(input: TokenStream) -> TokenStream
```

单独生成 `StructNameSet` 新类型 + 相关的 `SetWidgetSetExt`、`SetWidgetRefExt`、`SetWidgetExt` trait 和 `StructNameSetIterator`。

## 已注释的旧代码

文件开头和末尾保留了旧版 `#[derive(Widget)]` 宏的 `WidgetWrap` 实现以注释形式保留，不再使用但作为参考：

```rust
// 旧的 derive_widget 只生成 Widget trait（非 WidgetNode）
// 旧的 #[derive(WidgetWrap)] 是 WidgetNode 的前身
```

## 使用模式总结

```
┌─────────────────────────────────────────────────────┐
│                派生宏                               │
├──────────────┬──────────────────────────────────────┤
│ 应用开发:     │  #[derive(Widget)]                    │
│              │  推荐方式：一次性生成所有 trait         │
├──────────────┼──────────────────────────────────────┤
│ 精细控制:    │  #[derive(WidgetNode)]   (旧，已注释) │
│              │  #[derive(WidgetRef)]                  │
│              │  #[derive(WidgetRegister)]             │
│              │  #[derive(WidgetSet)]                  │
├──────────────┼──────────────────────────────────────┤
│ 动画支持:    │  #[derive(Animator)]                   │
└──────────────┴──────────────────────────────────────┘
```

**注意**：`WidgetNode` 的单独派生实现（`derive_widget_node`）已从 `#[proc_macro_derive]` 导出列表中移除（注释掉），仅通过主 `derive(Widget)` 宏间接使用。
