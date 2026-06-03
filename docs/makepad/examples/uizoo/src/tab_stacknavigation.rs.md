# tab_stacknavigation.rs

StackNavigation 栈式导航展示页面 — 演示页面推入/弹出操作。

## 按钮模板

```rust
let StackNavDemoButton = Button{
    width: Fit height: Fit
    padding: Inset{top: 10. bottom: 10. left: 20. right: 20.}
    margin: Inset{top: 5. bottom: 5.}
}
```

## StackNavigation 配置

```rust
stack_nav_demo := StackNavigation{
    width: Fill height: Fill
```

### 根视图

```rust
root_view +: {
    flow: Down  align: Align{x: 0.5 y: 0.3}  spacing: 10.  padding: 20.
    H3{text: "Root View"}
    Label{text: "This is the root of the StackNavigation."}
    push_view_a := StackNavDemoButton{text: "Push View A"}
    push_view_b := StackNavDemoButton{text: "Push View B"}
    push_view_c := StackNavDemoButton{text: "Push View C"}
}
```

三个按钮分别推入不同的视图。

### 子视图 A

```rust
stack_view_a := StackNavigationView{
    full_screen: false
    header +: { content +: { title_container +: { title +: {text: "View A"} } } }
    body +: {
        flow: Down  align: Align{x: 0.5 y: 0.3}  spacing: 10.  padding: 20.
        H3{text: "View A"}
        Label{text: "Use the back button (top-left) or mouse back to pop."}
        push_nested_from_a := StackNavDemoButton{text: "Push View B from here"}
    }
}
```

- `full_screen: false`: 在 Dock Tab 内原地导航（非全屏）。
- `header`: 顶部导航栏，包含返回按钮和标题。
- `body`: 主内容区，包含重直推入按钮。

### 子视图 B/C

类似结构，View C 包含 `pop_to_root_btn` 用于回到根视图。

## 事件处理（app.rs）

```rust
// 从根视图推入
if self.ui.button(cx, ids!(push_view_a)).clicked(&actions) {
    stack_nav.push(cx, live_id!(stack_view_a));
}
// 嵌套推入
if self.ui.button(cx, ids!(push_nested_from_a)).clicked(&actions) {
    stack_nav.push(cx, live_id!(stack_view_b));
}
// 回到根
if self.ui.button(cx, ids!(pop_to_root_btn)).clicked(&actions) {
    stack_nav.pop_to_root(cx);
}
```

API:
- `push(cx, view_id)`: 推入新视图（带动画）。
- `pop(cx)`: 返回上一级。
- `pop_to_root(cx)`: 回到根视图。
- 内置返回按钮处理 pop 操作。
