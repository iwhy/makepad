# slides/src/main.rs

演示 Makepad 的 `SlidesView` 组件，构建一个幻灯片演示应用。

## 整体结构

- **第7-208行**：`script_mod!` 定义全部内容和 UI
  - **第11-57行**：样式模板定义
    - `TalkSlide`：继承 `Slide`，设置内边距、背景色
    - `TalkChapter`：继承 `SlideChapter`，使用 theme 主色
    - `TalkBody` / `TalkSmall`：预定义的文本样式
  - **第59-208行**：UI 定义 — `SlidesView` 包含多个幻灯片（`TalkChapter` 和 `TalkSlide` 交替），每张幻灯片使用 `title.text` 设置标题，子元素使用 `TalkBody` 作为正文
- **第211-215行**：App 结构体
- **第217-226行**：`AppMain` — `handle_event` 直接将事件传递给 widget 树（SlidesView 处理键盘导航）

## 关键特性

- `SlidesView` 支持左右箭头键导航
- `TalkChapter` 和 `TalkSlide` 两种幻灯片类型，视觉风格不同
- `anim_speed: 0.86` 设置切换动画速度
- 内容完全在 DSL 中声明，无需 Rust 事件处理

## 设计模式

Slide/VerticalSplitter 模板的定义使用 `let` 关键字在脚本层声明，然后在 SlidesView 内直接实例化。每个幻灯片子元素通过 `title.text` 设置文本，使用 `TalkBody` 子组件呈现正文。
