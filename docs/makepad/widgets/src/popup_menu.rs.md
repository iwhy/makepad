# popup_menu.rs — 弹出菜单组件

## 概述
`PopupMenu` 是一个通用的弹出菜单/下拉菜单组件，支持文本项、分隔线、子菜单（嵌套菜单）和级联弹出。使用 `PopupMenuPosition` 控制菜单位置（相对于触发元素或固定位置），包含打开/关闭动画和悬停高亮。

## 核心结构

### PopupMenu
- **`draw_bg: DrawQuad`**：菜单背景绘制器（带阴影浮出效果）。
- **`list_walk: Walk`**：列表项的布局参数。
- **`menu_align: Align`**：菜单的对齐方式。
- **`separation: f64`**：菜单项之间的间距。
- **`animator: Animator`**：打开/关闭动画。
- **`text_style: TextStyle`**：菜单项文本样式。
- **`placement: Placement`**：菜单位置策略（托盘/固定/相对于矩形等）。
- **`menu_items: Vec<PopupMenuItem>`**：菜单项列表。
- **`keyboard_focus_index`/`pointer_hover_index`**：键盘/鼠标导航索引。
- **`selectable_items: Vec<WidgetRef>`**：可选择项引用列表。

### PopupMenuItem
本体枚举：`TextItem {text, accelerator, enabled, separator, submenu: Option<Vec<P>>}`、`CustomItem {widget}`。

### Placement
位置枚举：`Tray`（托盘对齐）、`Fixed`（固定偏移量）、`RelativeToRect`（相对于矩形）、`BelowInput`（文本输入框下方）、`Offset`（相对偏移）。

### PopupAction
菜单动作枚举：`Select{index, text}`（选中某菜单项）、`Close`（关闭菜单）。

## 核心方法

### Widget 实现

**`handle_event`**：分层处理事件。检测 `Event::Actions` 中的子菜单项选择（转发为自身 `Select` 动作）；处理键盘导航（方向键/回车/Escape）；处理鼠标悬停高亮和点击选择。非拖动鼠标点击菜单外部时发送 `Close` 动作并通过 `animator_reverse` 播放关闭动画。

**`draw_walk`**：调用 `draw_menu` 绘制菜单内容。

### 菜单绘制

**`draw_menu`**：按 Placement 策略计算菜单位置（Tray 模式根据窗口宽度左或右对齐；BelowInput 相对于输入框下方；Fixed/Offset/RelativeToRect 直接应用）。对每个 `PopupMenuItem` 绘制背景行、文本标签、加速键文本（靠右对齐）、指示有子菜单的 "›" 箭头、分隔线和自定义组件。延迟创建 FlatList 的模板项以提升性能。打开动画使用从零到一的缩放效果。

### 菜单管理

**`set_menu_items`**：设置菜单项列表，初始化键盘焦点索引，重置悬停索引。

**`find_item_by_action`**：递归搜索匹配指定动作的菜单项，返回 (索引, 项) 元组。

**`submenu_list`**：获取或创建子菜单组件。

### 显示/定位

**`popup_menu`**：静态方法，在给定位置显示弹出菜单。通过 `WidgetRef::new_from_widget` 创建 PopupMenu，设置位置、尺寸、菜单项后添加到窗口 root。

**`placement_for_rect`**：根据触发矩形和窗口尺寸计算最佳放置位置（下方/上方/左侧/右侧）。

**`calc_popup_pos`**：计算弹出菜单的原点坐标，考虑 Placement 类型和窗口边界，确保不超出窗口。

### 位置策略

**`tray_menu_pos`**：Tray 模式下根据鼠标在窗口中的位置决定菜单位置。

**`below_input_pos`**：BelowInput 模式下在输入框下方定位。

**`fixed_menu_pos`**：固定坐标菜单。

### MenuItems 辅助方法

**`text`/`name`/`submenu`**：访问 PopupMenuItem 的字段。

**`new_text_menu`/`new_separator`**：创建菜单项。

**`P::new_text_menu`/`P::new_separator`**：泛型便捷构造器。

## 输入弹出菜单

`InputPopup` 是 TextInput 专用的文本输入辅助弹出菜单（如文本选择、剪切/复制/粘贴操作），继承 `PopupMenu` 功能。

## 自定义绘图宏

`popup_menu_self_make_draw!` 宏简化了自定义 `draw_walk` 调用流程——它创建子作用域、执行 `draw_menu` 并在 `self.draw_bg` 上结束。
