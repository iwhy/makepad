# scroll_shadow.rs — ScrollShadow 滚动阴影指示器

## 文件概述

`ScrollShadow` 是在滚动视图边缘显示渐变阴影的装饰组件，提示用户内容在某个方向上还可以继续滚动。

---

## `ScrollShadow` 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct ScrollShadow {
    #[source] source: ScriptObjectRef,
    #[walk] walk: Walk,
    #[layout] layout: Layout,
    #[redraw] #[live] draw_shadow_top: DrawQuad,
    #[redraw] #[live] draw_shadow_bottom: DrawQuad,
    #[redraw] #[live] draw_shadow_left: DrawQuad,
    #[redraw] #[live] draw_shadow_right: DrawQuad,
}
```

四个 `DrawQuad` 分别对应上下左右四个方向的阴影。每个阴影是一个渐变矩形：

```
┌─┬───────┬─┐
│ │       │▲│ ← 顶部渐变
├─┤       ├─┤
│◄│ Content│►│
├─┤       ├─┤
│ │       │▼│
└─┴───────┴─┘
```

---

## 绘制逻辑

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    cx.begin_turtle(walk, self.layout);
    let rect = cx.turtle().rect();
    
    // 顶部阴影：从顶部边缘向下的渐变
    if self.scroll_y.scroll_pos > 0.0 {
        let top_rect = Rect::new(rect.pos, DVec2::new(rect.size.x, shadow_height));
        self.draw_shadow_top.draw_abs(cx, top_rect);
    }
    
    // 底部阴影：从底部边缘向上的渐变
    if self.scroll_y.scroll_pos < self.scroll_y.max_scroll_pos {
        let bot_rect = Rect::new(
            DVec2::new(rect.pos.x, rect.pos.y + rect.size.y - shadow_height),
            DVec2::new(rect.size.x, shadow_height)
        );
        self.draw_shadow_bottom.draw_abs(cx, bot_rect);
    }
    
    // 左右方向同理
    cx.end_turtle_with_area(&mut self.area);
    DrawStep::Done(self.area)
}
```

阴影的显隐逻辑：

1. **顶部阴影**：当 `scroll_pos > 0` 时显示，提示可以向上滚动。
2. **底部阴影**：当 `scroll_pos < max_scroll_pos` 时显示，提示可以向下滚动。
3. **左右阴影**：同理，根据水平滚动位置决定显隐。

---

## 渐变实现

每个 draw_shadow 的 shader 使用从透明到半透明的渐变纹理：

```
// 顶部阴影：从上到下渐变
pixel: fn() -> vec4 {
    let uv = self.pos.y;  // 0.0 (顶部) → 1.0 (底部)
    return vec4(0.0, 0.0, 0.0, mix(0.0, 0.3, 1.0 - uv));
}
```

通过 `mix(0.0, alpha, 1.0 - uv)` 实现从透明到半透明的渐变效果。

---

## 与 ScrollBar 的协作

ScrollShadow 通常与 ScrollBar 配合使用：

- ScrollBar 提供功能性的滚动控制（滑块、滚轮、拖拽）。
- ScrollShadow 提供视觉上的滚动边界提示。
- 两者都读取 `ScrollBar` 的 `scroll_pos` 和 `max_scroll_pos` 来控制行为。
