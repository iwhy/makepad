# scroll_bars.rs — ScrollBars 滚动条容器

## 文件概述

`ScrollBars` 是 ScrollView 的滚动管理容器，集成了一个或两个 `ScrollBar` 实例（横向 + 纵向），并管理滚动内容的偏移、裁剪和布局约束。

---

## `ScrollBars` 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct ScrollBars {
    #[source] source: ScriptObjectRef,
    #[deref] view: View,                // 继承 View 的容器能力
    #[live] animator: Animator,
    #[live] scroll_x: Option<ScrollBar>, // 水平滚动条
    #[live] scroll_y: Option<ScrollBar>, // 垂直滚动条
}
```

通过脚本 DSL 激活特定方向的滚动条：

```
mod.widgets.ScrollView = mod.widgets.ScrollBars {
    scroll_y := ScrollBar { orientation: Y }
}
```

---

## 核心功能

### 内容偏移管理

`ScrollBars` 负责将 `ScrollBar.scroll_pos` 转换为子组件绘制时的偏移量：

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    let offset_x = self.scroll_x.as_ref().map_or(0.0, |s| -s.scroll_pos);
    let offset_y = self.scroll_y.as_ref().map_or(0.0, |s| -s.scroll_pos);
    
    // 在 View 的布局中应用偏移
    cx.turtle().push_offset(DVec2::new(offset_x, offset_y));
    // 绘制子组件
    cx.turtle().pop_offset();
}
```

### 内容尺寸传递

`ScrollBars` 需要在绘制前将内容尺寸和可视区域尺寸设置到各个 ScrollBar：

1. 内容总尺寸由子组件布局计算得出。
2. 可视区域尺寸由 `walk` 约束和 `layout` 决定。
3. 绘制前通过 `scroll_x.set_content_size(content_width)` 和 `scroll_x.set_view_size(view_width)` 同步给 ScrollBar。

### 裁剪

绘制时应用裁剪（scissor rect）以防止子组件绘制到 ScrollBars 的可视区域之外：

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    let clip_rect = self.calculate_clip_rect();
    cx.turtle().push_clip(clip_rect);
    // 绘制带偏移的内容
    cx.turtle().pop_clip();
}
```

---

## 事件路由

`handle_event` 中先尝试将滚轮事件路由到 ScrollBar：

```rust
fn handle_event(&mut self, cx, event, scope) {
    if let Event::MouseScroll(e) = event {
        // 优先让 ScrollBar 处理滚动
        if let Some(bar) = &mut self.scroll_x {
            if bar.handle_event(cx, event, scope) == EventCapture::Sink {
                return EventCapture::Sink;
            }
        }
    }
    // 传递给子组件
    self.view.handle_event(cx, event, scope)
}
```

---

## 与 ScrollXView/ScrollYView 的关系

- `ScrollBars` 是底层实现，同时管理一个或两个方向的滚动条。
- `ScrollXView` / `ScrollYView` / `ScrollXYView` 是 `view_ui.rs` 中的便捷预设，内部使用 `ScrollBars`。
- 应用层通常使用 `ScrollYView` 等预设，不需要直接操作 `ScrollBars`。
