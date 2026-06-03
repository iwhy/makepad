# tab_portallist.rs

PortalList 高性能列表展示页面 — 演示虚拟化大量数据的列表渲染。

## 模板定义

```rust
let Post = View{
    width: Fill height: Fit
    padding: Inset{top: 10. bottom: 10.}
    body := RoundedView{
        width: Fill height: Fit
        content := View{
            width: Fill height: Fit
            text := P{text: ""}
        }
    }
}
```

列表项的 UI 模板：圆角卡片包裹文本内容。

## NewsFeed 组件

```rust
mod.widgets.NewsFeedBase = #(NewsFeed::register_widget(vm))
mod.widgets.NewsFeed = set_type_default() do mod.widgets.NewsFeedBase{
    list := PortalList{
        scroll_bar: ScrollBar{}
        TopSpace := View{height: 0.}      // 顶部间距模板
        BottomSpace := View{height: 100.}  // 底部间距模板
        Post := CachedView{                // 列表项模板，CachedView 缓存渲染结果
            flow: Down
            Post{}
            Hr{}
        }
    }
}
```

## NewsFeed Widget 实现

```rust
#[derive(Script, ScriptHook, Widget)]
struct NewsFeed {
    #[deref] view: View,
}
```

通过 `#[deref]` 委托给 `View`，实现 Widget trait。

### draw_walk

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep {
    while let Some(item) = self.view.draw_walk(cx, scope, walk).step() {
        if let Some(mut list) = item.borrow_mut::<PortalList>() {
            list.set_item_range(cx, 0, 1000);  // 1000 个虚拟项
            while let Some(item_id) = list.next_visible_item(cx) {
                let template = match item_id {
                    0 => live_id!(TopSpace),
                    _ => live_id!(Post),
                };
                let item = list.item(cx, item_id, template);
                // 根据 item_id % 4 生成不同内容的文本
                let text = match item_id % 4 {
                    1 => "At vero eos et accusam...",
                    2 => "How are you?",
                    3 => "Stet clita kasd gubergren...",
                    _ => "Lorem ipsum dolor sit amet...",
                };
                item.label(cx, ids!(content.text)).set_text(cx, &text);
                item.draw_all(cx, &mut Scope::empty());
            }
        }
    }
    DrawStep::done()
}
```

- `set_item_range(cx, 0, 1000)`: 声明 1000 个虚拟数据项（实际只渲染可见部分）。
- `next_visible_item`: 获取下一个可见项的 ID（PortalList 自动计算可见范围）。
- `list.item(cx, item_id, template)`: 按模板实例化列表项。
- `item_id == 0` 使用 `TopSpace` 模板，其余使用 `Post`。
- `item.label(ids!(content.text)).set_text(...)`: 更新具体子 widget 的文字。
- `item.draw_all(...)`: 绘制定位后的列表项。
