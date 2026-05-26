# `text_input.rs` — 文本输入框

## 作用
功能完整的单行/多行文本输入框，支持：
- 光标导航（键盘/鼠标/触摸）
- 文本选择（Shift + 方向键/鼠标拖拽）
- 剪贴板操作（复制/剪切/粘贴）
- 滚动（文本溢出时）
- 密码模式（字符遮蔽）
- IME 输入法集成
- 占位文本
- 输入掩码和过滤

## 自定义 Shader

### `DrawTextInputCursor`
- 基于 `DrawQuad` 扩展，添加 `cursor_offset`（光标闪烁动画）和 `cursor_color` 实例参数
- pixel shader：半透明光标 + 垂直渐变

## 关键结构

### `TextInputAction`
| 变体 | 说明 |
|------|------|
| `Return` | 回车确认 |
| `Up` / `Down` | 上下方向键（也适用于键盘导航选择） |
| `Tab` | Tab 切换焦点 |
| `Key { key: Key, mods: KeyModifiers }` | 原始按键事件（弹窗选择等场景） |
| `Changed` | 文本内容变化 |
| `Focused` / `Unfocused` | 焦点变化 |
| `SelectAll` | Ctrl+A 全选 |
| `None` | 无操作 |

### `TextInput`
| 字段 | 说明 |
|------|------|
| `draw_bg` | 背景四边形 |
| `draw_text` | 文本渲染 |
| `draw_cursor` | 光标渲染 |
| `draw_placeholder` | 占位文本渲染 |
| `text` | `Rc<RefCell<String>>` 文本缓冲区 |
| `max_indent` | 最大缩进（右边距） |
| `cursor_pos` / `selection_offset` | 光标和选区位置 |
| `left_scroll` | 水平滚动偏移 |
| `password` | `bool` 密码模式 |
| `next_frame` | 光标闪烁计时器 |
| `focused` | `bool` 是否获得焦点 |

## 方法详解

### 文本操作核心
- `get_text` / `set_text`：文本访问。设置文本时自动重设光标到末尾，重置左滚动，重绘
- `append_text`：追加文本并重绘
- `insert_string` / `remove_selection`：编辑操作
- `get_word_range` / `word_to_left`：单词级导航

### 滚动逻辑（`calc_left_scroll`）
- 根据光标位置计算水平滚动偏移，确保光标始终可见
- 采用 `look_ahead` 预读字符宽度防止光标贴边
- 使用 `SpaceNeededX` 几何查询计算字符位置

### 光标定位（`set_cursor_to_pos_x`）
- 点击坐标定位光标：从 `left_scroll` 偏移开始遍历字符，找到最近的字符插入位置
- 最后以字符中心补整（不是行号边缘）

### 选区管理
- `has_selection`：检查 `cursor_pos != selection_offset`
- `get_selection_range`：返回 `(min, max)` 有序范围
- `get_selected_text`：截取选区文本
- `select_all`：全选
- `delete_selection`：删除选中文本

### 焦点管理
- `set_key_focus` / `has_key_focus`：请求/检查键盘焦点
- `ignore_focus`：允许临时忽略焦点请求
- `set_select_all_on_focus`：获取焦点时自动全选

### `handle_event`（Widget）
- **鼠标事件**：`FingerDown` 定位光标、清除选区；`FingerMove` 扩展选区
- **剪贴板**：`TextCopy` 复制选中文本到系统剪贴板；`TextCut` 同上并删除；`TextPaste` 从剪贴板粘贴
- **键盘导航**：`Left`/`Right`/`Home`/`End`（支持 Shift 扩展选区）；`Up`/`Down` 触发行导航
- **编辑键**：`Backspace` 删除光标前字符；`Delete` 删除光标后字符；`Enter` 触发 `Return` action
- **IME 输入**：`CharInput` 处理字符输入（过滤不可见字符）；`IMEReplacement` 处理 IME 确认

### `draw_walk`（Widget）
- 根据焦点状态切换光标闪烁
- 光标闪烁使用 `next_frame` 定时器，每帧交替 `show_cursor` 状态
- 调用 `calc_left_scroll` 计算滚动偏移后再绘制文本

### `TextInputRef` 方法
- `text_input_action`：通用 action 检查
- `returned` / `changed` / `focused` / `unfocused` / `tab_pressed`：特定 action 检查
- `set_text_and_changed`：设置文本并强制触发 Changed action
- `center_cursor`：将光标居中到视口
