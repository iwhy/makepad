# stack_navigation.rs — StackNavigationView 页面导航栈

## 文件概述

`StackNavigationView` 实现了页面导航栈模式，类似 iOS `UINavigationController` 或 Android `FragmentManager` 的后退栈。支持通过 push/pop 操作切换页面，并带有滑动/淡入淡出过渡动画。

---

## `StackNavigationView` 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct StackNavigationView {
    #[source] source: ScriptObjectRef,
    #[deref] view: View,
    #[live] animator: Animator,
    #[live] transition: NavigationTransition,  // 过渡动画类型
    #[live] pages: Vec<WidgetRef>,             // 页面栈
    #[live] stack: NavigationStack,             // 栈状态管理
    #[rust] current_index: usize,
}
```

### `NavigationTransition`

```rust
pub enum NavigationTransition {
    None,              // 无过渡
    SlideHorizontal,   // 水平滑动
    SlideVertical,     // 垂直滑动
    Fade,              // 淡入淡出
}
```

### `NavigationStack`

```rust
pub struct NavigationStack {
    pages: Vec<NavigationPage>,  // 页面历史栈
}

pub struct NavigationPage {
    pub id: LiveId,           // 路由标识
    pub data: Option<WidgetRef>, // 关联的 Widget
}
```

---

## 导航操作

### push

```rust
fn push(&mut self, cx, page: WidgetRef, transition: Option<NavigationTransition>) {
    // 1. 保存当前页面状态
    // 2. 将新页面推入栈顶
    // 3. 播放入场动画（从右侧滑入 / 淡入等）
    // 4. 更新 current_index
}
```

Push 流程细节：

1. 当前页面进入"暂停"状态（不销毁，保持状态）。
2. 新页面被添加到 `pages` 数组末尾，`current_index += 1`。
3. 如果过渡不是 `None`，启动动画系统：
   - `SlideHorizontal`：新页面从右侧滑入，旧页面向左滑出。
   - `Fade`：旧页面淡出，新页面淡入。
4. 动画期间，两个页面同时绘制（过渡动画需要）。
5. 动画完成后，旧页面停止绘制。

### pop

```rust
fn pop(&mut self, cx, transition: Option<NavigationTransition>) -> Option<WidgetRef> {
    // 1. 从栈顶移除当前页面
    // 2. 恢复上一页面状态
    // 3. 播放退场动画（向左滑出 / 淡出等）
    // 4. 返回被 pop 的页面引用
}
```

Pop 流程：

1. 启动退场动画（与 push 方向相反）。
2. 上一页面从暂停状态恢复。
3. 动画完成后移除当前页面，`current_index -= 1`。
4. 如果栈中只有一个页面（根页面），pop 无效。

### popToRoot

```rust
fn pop_to_root(&mut self, cx) {
    // 依次 pop 直到只剩根页面
    // 或者使用单步动画直接回到根页面
}
```

---

## 过渡动画实现

以 `SlideHorizontal` 为例：

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    while self.view.draw_walk(cx, scope, walk).step() {
        // 1. 确定可见页面范围
        let (visible_pages, progress) = self.get_visible_pages();
        
        // 2. 按栈顺序绘制页面
        for (i, page) in visible_pages.iter().enumerate() {
            let offset = if i == visible_pages.len() - 1 {
                // 当前页面（顶部）：根据动画进度偏移
                DVec2::new(screen_width * (1.0 - self.anim_progress), 0.0)
            } else {
                DVec2::ZERO  // 底层页面不动
            };
            
            cx.turtle().push_offset(offset);
            page.draw_walk(cx, scope, walk);
            cx.turtle().pop_offset();
        }
    }
}
```

动画进度（`anim_progress`）由 Animator 控制，在 `frame_event` 中更新：
- `0.0`：动画开始（新页面在右侧完全不可见）。
- `1.0`：动画完成（新页面完全可见）。

---

## 页面生命周期

| 操作 | 当前页面 | 目标页面 |
|------|----------|----------|
| push | 暂停（停止事件处理） | 激活 |
| pop  | 销毁 | 激活（恢复事件处理） |
| popToRoot | 销毁 | 根页面激活 |

"暂停"状态：
- 页面仍然在内存中（`pages` 数组中）。
- 不处理事件（事件路由跳过）。
- 不绘制（除非在过渡动画中）。

---

## 与路由系统的关系

`StackNavigationView` 是底层导航组件，不直接处理路由解析。上层可以通过以下方式使用：

1. **直接调用**：`navigation.push(cx, page_widget_ref, None)`。
2. **通过 Action**：在 `handle_actions` 中解析路由事件并调用 push/pop。
3. **与 URL 路由集成**：将应用 URL 映射为页面 ID。

```rust
fn handle_actions(&mut self, cx, actions) {
    if self.navigation_view(id!(nav)).push_action(actions) {
        // 处理 push action
        let page = self.load_page(id!(settings_page));
        self.navigation.view(id!(nav)).push(cx, page, None);
    }
}
```
