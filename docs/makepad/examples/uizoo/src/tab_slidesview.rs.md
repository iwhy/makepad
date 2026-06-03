# tab_slidesview.rs

SlidesView 幻灯片展示页面。

```rust
SlidesView{
    width: Fill height: Fill

    SlideChapter{
        title := H1{text: "Hey!"}
        SlideBody{text: "This is the 1st slide. Use your right\ncursor key to show the next slide."}
    }

    Slide{
        title := H1{text: "Second slide"}
        SlideBody{text: "This is the 2nd slide. Use your left\ncursor key to show the previous slide."}
    }
}
```

- `SlidesView`: 幻灯片容器，支持键盘翻页（左右方向键）。
- `SlideChapter`: 章节幻灯片（通常带不同视觉风格）。
- `Slide`: 普通幻灯片。
- 每个幻灯片包含标题（H1）和正文（SlideBody）。
- 用于演示 Makepad 的演示文稿/幻灯片能力。
