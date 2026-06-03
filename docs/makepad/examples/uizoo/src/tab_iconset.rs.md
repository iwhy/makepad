# tab_iconset.rs

IconSet 字体图标展示页面。

```rust
mod.widgets.DemoIconSet = UIZooTabLayout_B{
    desc +: { Markdown{body: "# IconSet\n\nIconSet displays font-based icons."} }
    demos +: {
        flow: Right  spacing: 30.
        IconSet{text: "\u{f015}" draw_text +: {color: #0ff}}
        IconSet{text: "\u{f2bd}" draw_text +: {color: #0ff}}
        // ... 共 16 个字体图标
    }
}
```

- `IconSet` 使用字体字形（如 FontAwesome 的 Unicode Private Use Area 编码）显示图标。
- 每个 `IconSet` 的 `text` 属性设为 Unicode 码位（如 `\u{f015}` = fa-home）。
- `draw_text.color: #0ff` 统一设置为青色。
- `flow: Right spacing: 30.` — 水平排列，间距 30px。

展示的图标包括：home, user, image, file, camera, calendar, server, thumbs-up, smile, music, bell, envelope, code, database, etc.
