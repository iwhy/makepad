# command_text_input.rs — 命令式文本框（带弹出列表）

## 概述
`CommandTextInput` 是一个 `TextInput` 的包装器，在触发字符（如 `/`）输入时显示一个弹出选项列表。常用于命令面板、提及（@）补全等场景。支持内联搜索（inline_search）模式，其中搜索词直接在主输入框中输入。

## 核心结构

### CommandTextInput
- **`trigger`**：触发弹出列表的字符（如 `"/"`）。
- **`inline_search`**：是否在同一个输入框中进行搜索过滤。
- **`color_focus`/`color_hover`**：键盘焦点项/鼠标悬停项的高亮颜色。
- **`selectable_widgets`**：弹出列表中可选中的组件引用列表。
- **`keyboard_focus_index`/`pointer_hover_index`**：键盘导航和鼠标悬停的当前索引。
- **`trigger_position`**：触发字符在文本中的字节位置，用于内联搜索模式。

### InternalAction
内部动作枚举：`ShouldBuildItems`（需要重建列表项）和 `ItemSelected`（列表项被选中）。

### List
简化的列表组件，内部维护 `Vec<WidgetRef>`，在 `draw_walk` 中顺序绘制所有列表项。

## 核心方法

### Widget 实现

**`draw_walk`**：先调用 `update_highlights` 更新列表项的高亮颜色，再委托父 View 绘制。绘制后处理待定的焦点请求（搜索输入框或主输入框）。

**`handle_event`**：分层事件处理 — 首先处理键盘导航（方向键/回车/Escape）在弹出列表可见时吞噬事件；然后委托给父 View；接着处理文本输入事件检测触发字符；最后处理 `Event::Actions` 识别点击选择、悬停进入/离开、搜索框变化等。

### 弹出列表管理

**`show_popup`/`hide_popup`**：控制弹出列表的可见性。内联搜索模式下隐藏搜索输入框包装器。

**`clear_popup`**：清空触发器位置、搜索文本、光标和列表项。

**`reset`**：隐藏弹出并清空主输入框文本。

### 项目管理

**`clear_items`**：清空 List 中的项和 `selectable_widgets`。

**`add_item`**：向 List 添加可选择项，并初始化键盘焦点索引。

**`add_unselectable_item`**：添加不可选中的项（如分隔线、标题）。

### 触发与选择

**`on_text_inserted`**：检测输入文本的最后一个字素簇是否等于触发字符。如果是，显示弹出列表（内联模式隐藏搜索框包装器，非内联模式聚焦搜索输入框），发送 `ShouldBuildItems` 动作。

**`on_keyboard_controller_input_submit`**：键盘回车时选中当前键盘焦点项。

**`select_item`**：先调用 `try_remove_trigger_and_inline_search` 移除触发字符和搜索文本，再记录选中的组件，发送 `ItemSelected` 动作，隐藏弹出。

**`try_remove_trigger_and_inline_search`**：使用 Unicode 字素簇安全地移除触发字符及其后的内联搜索文本，重建输入文本并设置光标位置。

### 搜索文本

**`search_text`**：内联模式下从主输入框的触发字符后提取搜索文本；非内联模式下从搜索输入框获取。实现了 Unicode 字素簇级别的安全截取，支持 `MAX_SEARCH_TEXT_LENGTH`（100 字符）限制。

### 键盘导航

**`on_keyboard_move`**：按给定方向移动键盘焦点，处理边界情况（空列表、首尾循环）。

### 高亮更新

**`update_highlights`**：遍历所有可选择项，通过 `script_apply_eval!` 设置背景颜色 — 键盘焦点项使用 `color_focus`、鼠标悬停项使用 `color_hover`、默认项使用透明。键盘焦点优先于鼠标悬停。

### 辅助方法

**`ensure_popup_consistent`**：保持弹出列表状态一致性 — 内联搜索模式隐藏搜索输入框包装器，非内联模式显示。

**`trigger_grapheme`**：从 `trigger` 字符串中提取第一个字素簇。

**`key_controller_text_input_ref`**：根据内联模式返回主输入框或搜索输入框的引用。

### Unicode 辅助函数

**`graphemes`**：包装 `UnicodeSegmentation::graphemes(true)`。

**`get_head`**：获取文本框光标的字节索引。

**`is_whitespace`**：检查字素簇是否为空白。
