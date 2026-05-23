# html.rs — HTML 渲染组件

## 概述
`Html` 组件将 HTML 字符串解析为文档树并在 TextFlow 上渲染，支持标题（h1-h6）、段落、粗体/斜体/等宽、链接、列表（有序/无序）、表格、代码块、引用块、分隔线、上下标、`<details>/<summary>` 折叠展开等。

## 核心结构

### Html
- **`body`**：HTML 源字符串。
- **`doc`**：`HtmlDoc` 解析文档树。
- **`ul_markers`**：无序列表标记符号数组（按嵌套层级索引）。
- **`ol_markers`**：有序列表编号类型数组（`OrderedListType`）。
- **`list_stack`**：嵌套列表层级跟踪栈。
- **`details_stack`/`seen_details`/`skip_details_depth`**：`<details>` 折叠状态管理。
- **`summary_click_areas`/`summary_area_cache`**：`<summary>` 点击区域缓存。
- **`draw_summary_hit`**：覆盖在 `<summary>` 上的透明 DrawQuad，使整行可点击。

### HtmlLink
内联链接组件，通过 `scope.data` 访问父 HtmlDoc 提取 `href` 属性。支持三种交互状态（默认/悬停/按下）和三种点击动作：`Clicked`（左键/轻触）、`SecondaryClicked`（右键/长按），均携带 URL 和修饰键。

### 内部辅助类型
- **`DetailsLevel`**：记录 `<details>` 的 ID 和展开状态。
- **`ListLevel`**：列表层级状态，包含类型（有序/无序）、编号格式、当前计数和缩进。
- **`ListKind`**：`Unordered` 或 `Ordered`。
- **`OrderedListType`**：编号格式枚举（`Numbers`/`UpperAlpha`/`LowerAlpha`/`UpperRoman`/`LowerRoman`）。
- **`TrimWhitespaceInText`**：文本空白修剪策略（`Keep`/`Trim`）。

## 核心方法

### Widget 实现

**`draw_walk`**：主绘制入口。使用 `HtmlDoc::new_walker` 遍历文档节点树，处理 `<details>/<summary>` 的特殊展开/折叠逻辑，再分发到 `handle_open_tag`/`handle_close_tag`/`handle_text_node`。`<details>` 处于折叠状态时快速跳过其内容（`skip_details_depth` 计数器）。`<summary>` 绘制箭头按钮和文本后计算包围盒，覆盖透明点击区域。

**`handle_event`**：处理 `<summary>` 整行点击（路由到对应 FoldButton 切换）、监听 FoldButton 的 `Opening`/`Closing` 动作触发 TextFlow 重绘、委托给 TextFlow 处理选择等事件。

**`set_text`**：设置新 HTML 正文，重新解析文档，清空 `<details>` 状态缓存，触发重绘。

### HTML 标签处理

**`handle_open_tag`**：匹配开始标签名，调用 TextFlow 的对应方法设置样式。`h1-h6` 按比例缩放字号（2.0/1.5/1.17/1.0/0.83/0.67）；`p` 添加段落间距；`code` 缩小字号并开启等宽/内联代码；`pre` 进入代码块；`blockquote` 进入引用块；`ul/ol` 推入列表栈；`li` 生成标记文本（圆点或序号）；`table` 计算列数并开始表格；`tr/th/td` 处理表头/表体/单元格对齐。

**`handle_close_tag`**：匹配结束标签名，弹出样式栈，恢复字号，调用 TextFlow 对应结束方法。

**`handle_text_node`**：处理文本节点，根据 `TrimWhitespaceInText` 策略修剪空白。在表格内忽略全空白文本。

### 表格列数计算

**`count_table_columns`**：从指定索引后遍历节点，找到第一个 `<tr>` 中 `<td>`/`<th>` 的数量，处理嵌套表格深度。

### 链接处理

**`ScriptHook::on_after_new_scoped`**：HtmlLink 实例化后从作用域中的 HtmlDoc 提取 `href` 属性。

### HtmlLink 交互

**`handle_event`**：处理动画状态、鼠标/触摸命中测试 — `FingerDown` 触发按下动画，`FingerHoverIn` 设置手型光标和悬停动画，`FingerUp` 在按钮内部时发送 `Clicked` 动作并恢复悬停状态，长按/右键发送 `SecondaryClicked`。

### 自定义组件

**`handle_custom_widget`**：识别未知标签名，将其作为自定义模板组件在 TextFlow 中实例化。使用 `Scope::with_props_index` 传递文档索引以访问标签属性。

### 有序列表编号

**`OrderedListType::marker`**：根据计数和编号类型生成标记字符串（数字、字母、罗马数字）。支持负数回退到数字格式，罗马数字超过 3999 回退。

**`to_roman_numeral`**：将整数 1-3999 转换为大写罗马数字字符串。

### 单元格对齐

**`cell_align_x`**：从 HTML `style` 属性中的 `text-align` 声明或 `align` 属性解析水平对齐值（左侧 0.0、居中 0.5、右侧 1.0）。`style` 优先级高于 `align`。
