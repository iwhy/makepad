# tab_bar.rs — 标签栏组件

## 概述
`TabBar` 是 Makepad Studio 的标签栏容器，管理多个 `Tab` 的水平排列。支持标签溢出滚动、拖拽排序、新增标签按钮、左侧固定无标题区域和右侧溢出指示器按钮。当标签总宽度超出可视区域时，标签栏变为可滚动。

## 核心结构

### TabBar
- **`tabs_view: View`**：包含标签行的视图。
- **`add_tab_button: WidgetRef`**：新增标签按钮。
- **`scroll_to_side_button_left: WidgetRef`**：左滚动溢出指示器。
- **`scroll_to_side_button_right: WidgetRef`**：右滚动溢出指示器。
- **`scroll_offset: f64`**：当前水平滚动偏移（负数，向左滚动）。
- **`content_width: f64`**：所有标签内容总宽度。
- **`tabs_drag_index: u32`**：拖拽源标签索引。
- **`tabs_drag_pos: f64`**：拖拽时的滚动偏移。
- **`has_no_titlebar: bool`**：无标题栏模式（左上角不显示文件名）。

### TabBarItem
简单标签包装结构，内部包含 `Tab`。

## 核心方法

### Widget 实现

**`handle_event`**：处理标签滚动事件（`MouseScroll`/`Scroll` 水平滚动）、新增标签按钮点击、溢出按钮点击（每次点击滚动 300px）和兼容旧版 TabBar 的键盘方向键导航。

**`draw_walk`**：三阶段绘制。第一阶段绘制视图容器；第二阶段遍历所有标签子组件，计算各标签的 x 偏移量并绘制；第三阶段在右侧绘制新标签按钮。拖拽模式时特殊处理——计算标签放置位置，检测是否拖拽至左右侧边界（触发 `DragToSide` 动作）并自动滚动。

### 标签管理

**`tab_rect`/`tab_rect_px`**：计算指定索引标签的位置和大小。位置 = 所有前置标签宽度和 + scroll_offset。

**`draw_tab_bar_tab_item`**：根据标签索引和总标签数计算位置，应用 Walk 约束绘制标签项。

### 滚动控制

**`need_scroll`**：判断内容总宽是否大于可视区域宽度。

**`calc_scroll_offset`**：计算使得指定标签可见的最小滚动偏移量（左侧容器不足时滚动使标签可见）。

**`scroll_to_tab`**：滚动标签栏使指定索引标签可见。

**`scroll_amount`**：限制滚动偏移量范围（最大 0，最小为 `可视宽度 - 内容宽度`）。

### 溢出指示器

**`scroll_overflow_left`/`scroll_overflow_right`**：根据滚动偏移和内容宽度判断左右侧是否溢出。

**`update_scroll_side_buttons`**：根据溢出状态显示/隐藏左右溢出按钮。

### 拖拽

**`drag_end`**：拖拽结束时更新滚动偏移，计算目标位置，发送 `DragEnd` 动作携带新旧索引。

**`calculate_drag_target`**：根据鼠标 x 位置计算拖拽的目标索引（落在标签左半部分 → 左侧，右半部分 → 右侧）。

### TabItem

`TabItem` 递归委托给内部 `Tab` 的所有方法（`handle_event`、`draw_walk`、`tapped`等），添加了 `tab_file_path_str` 和 `set_tab_file_path_str` 用于脚本层访问文件路径。

### TabBarRef

`TabBarRef` 提供对 TabBar 子组件的安全访问：`tab`（获取标签）、`add_tab_button`、`set_tab_count`、`update_item_count`（更新标签数量标签同步关闭）。
