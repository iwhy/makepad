# `lib.rs` — 小部件库入口和注册

## 作用
Makepad 小部件库（`makepad-widgets` crate）的入口文件。负责统一注册所有小部件，导出预定义模块（theme、draw、widgets），并提供数据结构可视化调试工具。

## 结构详解

### 模块组织
```
widgets/src/
├── lib.rs               # 入口：注册所有小部件，组织 module tree
├── label.rs             # Label/H1/H2/H3/LinkLabel
├── button.rs            # Button/ButtonFlat/ButtonFlatter
├── checkbox.rs          # CheckBox/Toggle
├── radio.rs             # RadioButton/RadioGroup
├── text_input.rs        # TextInput（单行/多行文本输入）
├── slider.rs            # Slider
├── dropdown.rs          # DropDown
├── image.rs             # Image
├── image_blend.rs       # ImageBlend（交叉渐变动画）
├── image_cache.rs       # ImageFit / ImageCache 重导出
├── icon.rs              # Icon
├── animated_image_gif.rs # AnimatedImageGif
├── loading_spinner.rs   # LoadingSpinner
├── scroll_bars.rs       # ScrollBar/ScrollBars
├── portal_list.rs       # PortalList（虚拟滚动列表）
├── flat_list.rs         # FlatList
├── handle_tree.rs       # HandleTree（可拖拽分隔条）
├── splitter.rs          # Splitter
├── dock.rs              # Dock（标签页面板系统）
├── navigation.rs        # StackNavigation/ExpandablePanel/NavBar
├── modal.rs             # Modal
├── tooltip.rs           # Tooltip
├── popup_notification.rs # PopupNotification
├── file_tree.rs         # FileTree
├── page_flip.rs         # PageFlip
├── cached_widget.rs     # CachedWidget
└── html.rs / markdown.rs #（可选功能，已 gate 关闭）
```

## 宏 `self::widget_module!` — 核心注册函数

```rust
pub fn script_mod(vm: &mut ScriptVm) {
    register_all_widgets(vm);  // 注册所有小部件
    register_all_draw(vm);     // 注册所有自定义绘制 shader
    register_all_theme(vm);    // 注册主题变量
}
```

### 预定义模块

#### `mod.prelude.widgets` — 应用开发预定义
- 导出了所有标准小部件名称，供应用 `use mod.prelude.widgets.*`
- 包含别名 `theme:mod.theme`、`draw:mod.draw`
- 包含 `mod.widgets.*` 中的所有小部件

#### `mod.prelude.widgets_internal` — 内部开发预定义
- 用于小部件库本身的内部构建
- 包含 `mod.widgets.*` 中所有已注册的小部件
- 不包含用户级 theme/draw 别名

### 所有小部件的注册顺序
在 `widget_module!` 宏中，小部件按以下顺序注册到 `mod.widgets`：

1. 基础控件：View、Label（及 H1/H2/H3）、LinkLabel、Button/ButtonFlat/ButtonFlatter
2. 输入控件：CheckBox、Toggle、TextInput、Slider、DropDown
3. 单选控件：RadioButton、RadioGroup
4. 媒体控件：Image、ImageBlend、Icon、AnimatedImageGif、LoadingSpinner
5. 滚动控件：ScrollBar、ScrollBars
6. 列表控件：PortalList、FlatList
7. 布局控件：HandleTree、Splitter
8. Dock 系统：DockTabs、DockSplitter、Dock
9. 导航控件：StackNavigation、ExpandablePanel
10. 覆盖层控件：Modal、Tooltip、PopupNotification
11. 文件树：FileTree
12. 页面控件：PageFlip
13. 缓存控件：CachedWidget

## LiveId 常量

`lib.rs` 定义了整个库中使用的所有 LiveId 常量：

### 通用 ID
- `main_window`、`body`、`content`、`root`

### 文本输入相关
- `text_input`、`input`、`search_input`、`placeholder_input`、`search`

### 按钮相关
- `button`、`my_button`、`close_button`、`go_back`、`ok`、`cancel`

### 模态/弹窗相关
- `modal`、`bg_area`、`bg_view`、`tooltip`、`popup`、`popup_notification`

### 列表/树相关
- `list`、`portal_list`、`flat_list`、`item`、`header`、`title`、`row`、`tree`

### 面板相关
- `panel`、`panel_left`、`panel_right`、`main_panel`、`side_panel`、`top_panel`、`bottom_panel`、`center_panel`、`splitter`

### 文本/标签相关
- `label`、`tooltip_label`、`description`、`label_kind`、`label_text`、`label_icon`、`link_label`、`fold_header`、`nav_bar`

### 图标相关
- `icon`、`lock_icon`、`caret`、`chevron`

### 滚动相关
- `scroll_bar`、`scroll_bars`、`scrollbars`

### 编辑/代码相关
- `editor`、`code_block`、`code_editor`、`tab_strip`、`tabs`、`handle`

### 其他控件
- `slider`、`selection`、`search_results`、`progress`、`status`、`fold_button`、`horizontal_handle`、`vertical_handle`、`image`、`profile`、`menu`、`picker`、`color_picker`、`file_tree`、`file_list`、`sidebar`、`new_tab`、`expandable`

### 触发 ID（events/actions 中使用的标记）
- `drop`、`context_menu`、`return`、`changed`、`pressed`、`focused`、`unfocused`
- `start_drag`、`end_drag`、`drag`、`drag_begin`、`drag_end`、`click`
- `clicked`、`selected`、`tab_selected`、`tab_closed`、`dismissed`
- `new_tab_button`、`debug_inspect`、`open_file`、`close_document`
- `scroll_to_show`、`clear_builds`、`clear_build`、`scroll_to`、`scroll_to_item`
- `SmoothScrollReached`、`BackPressed`、`new_default_event`

### 尺寸/动画常量
- `DEFAULT_HEIGHT`：默认行高 22
- `DEFAULT_MIN_DRAG_DISTANCE`：最小拖拽距离 5（px）

## `cargo build` 构建标记（feature gates）
- `html`：启用 HTML 渲染支持
- `markdown`：启用 Markdown 渲染支持
- `document`：启用文档模式
- `network`：启用网络功能
- `web`：启用 Web 平台支持
- `image_loading`：启用图像加载功能
- `default_build_image`：默认构建镜像

## 关键设计说明

`lib.rs` 是小部件库的核心注册枢纽；

### 数据可视化：`debug_data_tree`

该函数递归遍历任意小部件的内部字段，输出结构化的调试信息。它检查字段是否被标记为 `#[live]`、`#[walk]` 等，为开发者提供一个树状的视图来检查小部件的内部状态。输出格式：
- `[widget] struct_name`：小部件类型
- `#[live]` 标记的字段显示当前值
- `#[rust]` 字段仅在非默认值时显示
- 递归遍历子控件结构

### 注册流程
1. `crate::script_mod(vm)` 被应用的 `App::run()` 或 Studio 启动时调用
2. 内部调用 `register_all_widgets(vm)` 逐一注册每个小部件到 `mod.widgets`
3. 然后注册自定义 draw shader 到 `mod.draw`
4. 最后注册 theme 变量到 `mod.theme`
5. 应用端的 `script_mod!` 通过 `use mod.prelude.widgets.*` 使用已注册的名称
