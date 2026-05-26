# text_flow.rs — 富文本流布局引擎

## 概述
`TextFlow` 是 Makepad 框架的核心富文本排版组件，支持段落流式布局、内联格式（粗体/斜体/等宽/下划线/删除线）、代码块、引用块、列表、表格、链接、内联代码高亮、文本选择、流式打字动画等功能。

## 核心结构

### FlowBlockType (枚举)
定义文本流中可绘制的块类型：`Quote`(引用)、`Sep`(分隔线)、`Code`(代码块)、`InlineCode`(内联代码)、`Underline`(下划线)、`Strikethrough`(删除线)、`Selection`(选择高亮)、`TableCell`(表格单元格)。每种类型对应 shader 中不同的 SDF 绘制路径。

### DrawFlowBlock
继承自 `DrawQuad` 的绘制类型，包含行颜色、分隔线颜色、代码背景色、引用前后景色、选择高亮色、表格边框色等实例字段。shader `pixel` 函数根据 `block_type` 匹配分支绘制不同视觉效果。

### SelectionTracker & SelectionSegment
选择跟踪器维护 `segments: Vec<SelectionSegment>` 和累积文本 `text: String`。`SelectionSegment` 有三种变体：
- **Text**：带排版缓存的文本段，记录 `laidout_text`、屏幕坐标原点、字体缩放和文本起始索引。
- **Gap**：非文本间隙（如图标、内嵌组件），记录包围盒和文本起始。
- **WidgetText**：子组件（如 CodeView）的文本区域，组件自行绘制选择高亮，TextFlow 跳过绘制。

### RectAreasTracker
区域追踪器用于收集绘制过程中产生的字符运行矩形区域，支持 push/pop 栈操作，供链接点击测试使用。

## 核心方法

### Widget 实现

**`draw_walk`**：两阶段绘制 — 第一阶段调用 `begin(cx, walk)` 初始化海龟布局、清空样式栈、重置字符索引；第二阶段调用 `end(cx)` 刷新延迟排版段、绘制选择高亮矩形、结束海龟并将项目保留可见。支持流式动画更新。

**`handle_event`**：先处理子组件事件和流式动画 `NextFrame`；再处理选择交互 — `FingerDown` 设置选择锚点，`FingerMove` 扩展选择范围并传播到子组件，`FingerUp` 显示剪贴板操作。支持 `TextCopy`/`TextCut`/`Ctrl+A` 全选。

### 布局控制

**`begin`/`end`**：初始化海龟、清空所有样式栈（粗体/斜体/等宽/下划线/删除线/内联代码/字号/颜色/换行合并等）、重置行计数和截断标记。`end` 时刷新排版并触发流式动画下一帧。

**`draw_text`**：核心文本绘制入口。根据当前样式栈选择合适的 TextStyle（bold/italic/fixed 组合），应用字号/颜色/纵移量/对齐。支持 `max_lines` 行数限制和 `TextOverflow` 截断。内联代码模式下特殊处理：膨胀 turtle 内边距为代码框留出空间，处理换行避免孤立的空框，绘制代码框背景后再恢复内边距。

### 块级元素

**`begin_code`/`end_code`**：设置 `block_type = Code`，在代码布局中绘制圆角矩形背景。`end` 时恢复 area 栈。

**`begin_quote`/`end_quote`**：设置 `block_type = Quote`，绘制引用块左侧竖线和高亮背景。

**`begin_list_item`/`end_list_item`**：绘制列表项标记（如圆点或数字），计算基于字号的缩进，调整左内边距使换行文本对齐标记后的内容。

**`sep`**：绘制水平分隔线。

### 表格

**`begin_table`/`end_table`/`begin_table_row`/`end_table_row`/`begin_table_cell`/`end_table_cell`**：完整表格布局，支持表头/表体、列数计算、单元格宽度均分、水平对齐（left/center/right）。`draw_row_cell_borders` 在行布局完成后绘制单元格边框（包含表头背景色、首行上边框、首列左边框）。

### 模板管理

**`item_with`/`item`/`item_counted`/`existing_item`/`clear_items`**：按模板 ID 创建或复用子组件，支持模板热替换（重新创建组件）。`item_with_scope` 使用作用域创建，用于 HTML 链接等需要访问属性的场景。

**`apply_template`**：将模板对象存入 `templates` 映射，并对已存在的匹配项重新应用模板。

### 流式动画

**`start_streaming_animation`/`reset_streaming_animation`/`stop_streaming_animation`**：控制文本渐入动画。动画机制使用比例追赶算法：每帧追赶剩余距离的 15%（基准 60fps 缩放），加上最小速度确保持续推进。`animated_chars` 滞后于 `actual_chars` 产生渐入效果。

### 选择功能

**`set_selection`/`clear_selection`/`select_all`/`selected_text`/`get_text_for_range`/`get_full_text`**：完整的选择管理。`propagate_selection_to_children` 将子范围传播给 WidgetText 组件（如 CodeView）。`selection_rects` 将字符范围转换为屏幕选择矩形列表（含 2px 下延填充）。

**`draw_selection_rects`**：遍历选择矩形并调用 `draw_selection.draw_abs` 绘制高亮。

### 辅助

**`walk_margin`**：在 turtle 中行走固定宽度的零高度矩形，用于添加间距。

**`draw_link`**：绘制可点击链接。创建计数器 ID 的组件实例，设置文本和动作数据后绘制。

### SelectionTracker 辅助方法

**`point_to_index`**：将屏幕坐标转换为字符索引。遍历所有段：Text 段使用排版器的 `point_in_lpxs_to_cursor`；Gap 段检查包围盒；WidgetText 段按 y 位置线性插值。如果点在所有段外部，回退到 `nearest_index`。

**`nearest_index`**：计算点到各段包围盒的最小距离，返回最近段的字符索引。

**`point_to_rect_distance`**：计算点到轴对齐矩形的最短欧几里得距离。

### TextFlowLink

链接组件通过 `scope.data` 访问父 TextFlow，在绘制时压入下划线样式和颜色，记录 `drawn_areas` 用于命中测试。支持 hover/down 状态颜色切换。
