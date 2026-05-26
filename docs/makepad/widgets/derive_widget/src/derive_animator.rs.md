# derive_animator.rs — Animator 派生宏实现

## 概述

`derive_animator.rs` 是 `#[derive(Animator)]` 派生宏的核心实现。它为带有 `animator` 字段的结构体自动生成 `AnimatorImpl` trait 的实现代码，使得该结构体能够将动画状态变化通过 `script_apply` 应用到脚本层。

## 支持的属性

- **`#[animator]`** — 标记在结构体字段上，指示该字段是 `Animator` 类型的动画控制器。
- **`#[source]`** — 标记在脚本对象引用字段上（`ScriptObjectRef`），用于脚本属性应用。如果缺少此属性，宏会生成编译错误：
  ```
  compile_error!("Animator derive requires a field with #[source] attribute to hold the ScriptObjectRef");
  ```

## 触发方式

在 `lib.rs` 中注册为：
```rust
#[proc_macro_derive(Animator, attributes(animator,))]
pub fn derive_animator(input: TokenStream) -> TokenStream {
    derive_animator_impl(input)
}
```

## 生成的代码

对于以下结构体：
```rust
#[derive(Animator)]
pub struct MyWidget {
    #[source] source: ScriptObjectRef,
    #[apply_default] animator: Animator,
    // ... 其他字段
}
```

宏生成如下 `AnimatorImpl` 实现（以下为示意代码）：

```rust
impl AnimatorImpl for MyWidget {
    fn animator_play_scoped(&mut self, cx: &mut Cx, state: &[LiveId; 2], play: Option<Play>, scope: &mut Scope) {
        if let Some(value) = self.animator.play(cx, state, play) {
            cx.with_vm(|vm| self.script_apply(vm, &Apply::Animate, scope, value));
        }
    }

    fn animator_in_state(&self, cx: &Cx, check_state_pair: &[LiveId; 2]) -> bool {
        self.animator.in_state(cx, check_state_pair)
    }

    fn animator_cut_scoped(&mut self, cx: &mut Cx, state: &[LiveId; 2], scope: &mut Scope) {
        if let Some(value) = self.animator.cut(cx, state) {
            cx.with_vm(|vm| self.script_apply(vm, &Apply::Animate, scope, value));
        }
    }

    fn animator_handle_event_scoped(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope) -> AnimatorAction {
        let mut act = AnimatorAction::None;
        if let Some(value) = self.animator.handle_event(cx, event, &mut act) {
            cx.with_vm(|vm| self.script_apply(vm, &Apply::Animate, scope, value));
        }
        act
    }
}
```

### 四个 trait 方法说明

| 方法 | 功能 |
|------|------|
| `animator_play_scoped` | 播放/切换动画状态，通过 `script_apply` 将动画值应用到脚本对象 |
| `animator_in_state` | 检查当前是否处于指定状态对（state pair） |
| `animator_cut_scoped` | 立即跳转到目标状态（无动画过渡），应用结果到脚本对象 |
| `animator_handle_event_scoped` | 处理动画事件（如鼠标悬停引起的状态切换），返回 `AnimatorAction` |

所有方法都将 `Animator` 输出的值通过 `cx.with_vm(|vm| self.script_apply(...))` 应用到脚本层，实现数据驱动 UI 更新。

## 核心数据结构和模式

- **`TokenBuilder`** (`tb`) — 代码生成器，用于逐步构造输出 TokenStream
- **`TokenParser`** (`parser`) — 输入解析器，用于解析 Rust 源代码的 TokenStream
- **`eat_attributes()`** — 消耗属性列表（`#[...]`），此处未使用返回值
- **`eat_ident("pub")`** / `eat_ident("struct")` — 顺序解析 pub struct 声明
- **`expect_any_ident()`** — 获取结构体名称
- **`eat_generic()`** — 解析泛型参数
- **`eat_all_types()`** / `eat_all_struct_fields()` — 解析类型约束（此处 expect 无类型形式）和字段定义
- **`eat_where_clause()`** — 解析 where 子句
- **`error()`** — 生成编译错误

## 处理流程

1. 解析 `pub struct StructName<Generics> { fields }` 结构
2. 在所有字段中查找名为 `animator` 的字段（通过字段名匹配，不是通过属性）
3. 检查是否存在带 `#[source]` 属性的字段；如果不存在，生成编译错误
4. 为结构体生成 `AnimatorImpl` 实现，所有方法均委托给 `animator` 字段

## 注意事项

- 宏假设字段名为 `animator`，不通过属性标记查找，而是通过字段名称直接匹配
- `#[source]` 属性是强制性的，缺少时会生成编译错误而非 panic
- 宏支持泛型参数和 where 子句，但要求结构体没有额外的类型表达式形式
- 生成的代码中 `script_apply` 的 `Apply::Animate` 表明这是动画驱动的属性更新
