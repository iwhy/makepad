# `file_tree.rs` — 文件树（核心组件）

## 作用
实现一个可展开/折叠的文件树控件，支持文件/文件夹图标、Git 状态指示点、选中/悬停/焦点动画、拖拽操作和奇偶行交替颜色。

## 自定义绘制 Shader

### `DrawBgQuad`
- 扩展 `DrawQuad`，添加 `is_even`、`scale`、`is_folder`、`focussed`、`active`、`hover`、`opened` 实例参数
- pixel shader 中通过混合 `color_1`/`color_2`（奇偶行）和 `color_active`（选中态）实现行高亮

### `DrawNameText`
- 扩展 `DrawText`，添加 `color_active` 等相同实例参数

### `DrawIconQuad`
- 扩展 `DrawQuad`，添加相同实例参数 + `is_folder` 字段
- pixel shader 中使用 SDF 绘制文件夹图标（两个矩形 union）

### `DrawStatusDotQuad`
- 扩展 `DrawQuad`，添加 `status_kind`（`GitStatusDotKind` 枚举）
- 支持四种 Git 状态：`New`（绿色）、`Modified`（橙色）、`Deleted`（红色）、`Mixed`（红色）
- 使用 SDF circle + 混合实现圆点

### `GitStatusDotKind`
- `None`、`New`、`Modified`、`Deleted`、`Mixed`，映射到 shader 中的颜色选择

## 关键结构

### `FileTreeNode`
| 字段 | 类型 | 说明 |
|------|------|------|
| `draw_bg` | `DrawBgQuad` | 行背景 |
| `draw_icon` | `DrawIconQuad` | 图标 |
| `draw_status_dot` | `DrawStatusDotQuad` | Git 状态点 |
| `draw_text` | `DrawNameText` | 节点文本 |
| `animator` | `Animator` | hover/focus/select/open 动画 |
| `is_folder` | `bool` | 是否为文件夹 |
| `indent_width` / `indent_shift` | `f64` | 缩进控制 |
| `opened` / `focussed` / `hover` / `active` | `f32` | 动画状态 |

### `FileTree`
| 字段 | 类型 | 说明 |
|------|------|------|
| `scroll_bars` | `ScrollBars` | 滚动条 |
| `file_node` / `folder_node` | `ScriptObjectRef` | 文件/文件夹模板 |
| `filler` | `DrawBgQuad` | 空白填充行 |
| `node_height` | `f64` | 节点高度 |
| `tree_nodes` | `ComponentMap<LiveId, FileTreeNode>` | 节点实例 |
| `open_nodes` | `HashSet<LiveId>` | 打开的文件夹 |
| `selected_node_id` | `Option<LiveId>` | 选中节点 |
| `stack` | `Vec<f64>` | 递归缩进/折叠状态栈 |

## 方法详解

### `FileTreeNode::set_draw_state`
- 同步设置 `draw_bg`、`draw_text`、`draw_icon` 的 `scale`、`is_even`、`is_folder` 状态

### `FileTreeNode::draw_folder`
- 使用 `Walk::new(Fill, Fixed(height))` 开始绘制背景
- 调用 `indent_walk` 进行缩进，可选绘制 Git 状态点
- 绘制文件夹图标 + 文字

### `FileTreeNode::draw_file`
- 与 `draw_folder` 类似，但始终显示状态点

### `FileTreeNode::indent_walk`
- 计算缩进宽度：`depth * indent_width + indent_shift`，如果状态点在缩进区域内则回收其宽度
- 返回带 margin 的 `Walk`

### `FileTreeNode::handle_event`
- 处理 hover 进出、手指移动（拖拽检测 `>= min_drag_distance`）、手指按下（切换选中态，切换文件夹展开/折叠）

### `FileTree::begin`
- 启动 `scroll_bars`，重置 `count`（行计数器）

### `FileTree::end`
- 用 `filler` 填充剩余空白区域，维持奇偶交替色
- 绘制滚动阴影
- 只保留选中的节点（其他不可见节点清除）

### `FileTree::should_node_draw`
- 检查当前节点是否在可见区域内（通过 `walk_turtle_would_be_visible`）
- 不可见时仍然 walk 但返回 false，维持布局一致性

### `FileTree::begin_folder_with_status`
- 递增行计数，调用 `should_node_draw` 决定是否实际绘制
- 从 `tree_nodes` 获取或创建节点实例，调用 `draw_folder`
- 将 `opened` 值 × scale 压入 stack 控制子节点可见性
- 如果 `opened <= 0.001`（折叠动画结束），立即调用 `end_folder` 跳过子节点

### `FileTree::file_with_status`
- step 同上，绘制文件节点

### `FileTree::set_folder_is_open`
- 更新 `open_nodes` 集合，调用节点的 `set_folder_is_open`

### `FileTree::start_dragging_file_node`
- 通过 `cx.start_dragging` 启动平台拖拽

### `handle_event`（Widget）
- 遍历所有节点，收集 `FileTreeNodeAction`
- 处理 Open/Close/Click/Drag 动作
- 对选中节点设置/取消 focus 状态
- 生成 `FileTreeAction::FileClicked` / `FolderClicked` / `ShouldFileStartDrag`

### `draw_walk`（Widget）
- 两阶段绘制模式：`begin` → `draw_state` → `end`

### `FileTreeRef` 方法
- `file_clicked` / `folder_clicked`：检查 action 中是否有文件/文件夹点击事件
- `set_folder_is_open` / `should_file_start_drag` 等操作代理
