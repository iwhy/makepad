# tab_pageflip.rs

PageFlip 页面翻转展示页面。

```rust
mod.widgets.DemoPageFlip = UIZooTabLayout_B{
    desc +: { Markdown{body: "# PageFlip\n\nPageFlip switches between pages."} }
```

### 导航按钮

```rust
pageflipbutton_a := Button{text: "Page A"}
pageflipbutton_b := Button{text: "Page B"}
pageflipbutton_c := Button{text: "Page C"}
```

三个按钮分别在 app.rs 中绑定事件：
```rust
self.ui.page_flip(cx, ids!(page_flip)).set_active_page(cx, live_id!(page_a));
self.ui.page_flip(cx, ids!(page_flip)).set_active_page(cx, live_id!(page_b));
self.ui.page_flip(cx, ids!(page_flip)).set_active_page(cx, live_id!(page_c));
```

### PageFlip 内容

```rust
page_flip := PageFlip{
    width: Fill height: Fill
    flow: Down
    active_page: @page_a

    page_a := View{ show_bg: true draw_bg +: {color: uniform(#f00)} H3{text: "Page A"} }
    page_b := View{ show_bg: true draw_bg +: {color: uniform(#080)} H3{text: "Page B"} }
    page_c := View{ show_bg: true draw_bg +: {color: uniform(#008)} H3{text: "Page C"} }
}
```

- `active_page: @page_a`: 初始显示 A 页。
- 三个页面各有不同背景色（红/绿/蓝），标题居中显示。
- 点击按钮可跳转到对应页面，PageFlip 提供切换动画。
