# lib.rs

库入口文件 — 声明所有公开模块。

```rust
pub use makepad_widgets;
pub mod app;
pub mod demofiletree;
pub mod layout_templates;

pub mod tab_adaptiveview;
pub mod tab_align_scroll;
pub mod tab_button;
// ... 共 30+ 个 tab_* 模块 ...
pub mod tab_widgetsoverview;
```

- `pub use makepad_widgets`: 重导出 widgets 库，使其他模块可以通过 `crate::makepad_widgets` 访问。
- `pub mod app`: 应用主模块。
- `pub mod demofiletree`: 自定义文件树组件模块。
- `pub mod layout_templates`: 布局模板（`UIZooTabLayout_B`、`UIZooRowH`）。
- 所有 `tab_*` 模块: 各个 Widget 的 Demo 展示模块，每个对应一个 Dock Tab。
