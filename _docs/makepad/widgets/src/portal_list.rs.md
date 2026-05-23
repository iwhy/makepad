# `portal_list.rs` — 虚拟滚动列表（核心组件）

## 作用
实现高效的虚拟化长列表，只渲染可见区域内的项。支持拖拽滚动、惯性滑动、顶部回弹、平滑滚动到指定项、尾部自动跟踪、跨项文本选择、像素级滚动条。是 Makepad 最复杂和核心的控件之一。

## 核心数据结构和枚举

### `ScrollState` — 滚动状态机
| 变体 | 说明 |
|------|------|
| `Stopped` | 停止状态 |
| `Drag { samples, initial_abs, committed }` | 手指拖拽中，sampled 保存速度采样点 |
| `Flick { delta, next_frame }` | 惯性滑动，delta 为每帧偏移量 |
| `Pulldown { next_frame }` | 顶部回弹动画 |
| `ScrollingTo { target_id, delta, next_frame, top_offset }` | 平滑滚动到指定项 |
| `Tailing { next_frame, velocity }` | 弹簧阻尼尾部跟踪动画 |

### `HeightTree` — Fenwick 树（二叉索引树）
- 用于 O(log n) 前缀和查询，将虚拟滚动位置映射到具体项索引
- 支持增量更新（单项高度变化时只更新 O(log n) 个节点）
- 通过 `find_position(target)` 实现二分查找：给定像素偏移，返回 (item_index, offset_within_item)
- `resize` 方法在范围变化时智能扩展（新增项只需追加）或重建（缩小）
- `update_default_height` 在平均高度变化时批量更新所有未测量项

### `ListDrawState` — 绘制状态机
- `Begin` → `Down`（正向遍历）→ 可能切换到 `Up`（反向遍历顶部可见项）→ `DownAgain`（回到正向）→ `End`

### `HeightCache` — 平均高度缓存
- 记录已测量项的高度总和和数量，用于对新项做默认高度估计

## 关键字段

| 字段 | 说明 |
|------|------|
| `range_start` / `range_end` | 数据范围（半开区间） |
| `first_id` / `first_scroll` | 当前视口顶部第一个项 ID 及其滚动偏移 |
| `view_window` | 可见窗口大小，用于估算 |
| `templates` | `HashMap<LiveId, ScriptObjectRef>`，模板定义 |
| `items` | `ComponentMap<usize, WidgetItem>`，当前活跃的项 |
| `reusable_items` | 缓存池，项滚出视口后回收入池 |
| `height_tree` | `Option<HeightTree>`，像素级高度追踪树 |
| `tail_range` / `auto_tail` / `smooth_tail` | 尾部自动跟踪相关 |
| `at_end` / `not_filling_viewport` | 视口填充状态 |
| `selectable` / `selection_anchor` / `selection_cursor` | 跨项文本选择 |

## 方法详解

### `ScriptHook` 实现
- `on_before_apply`：重载时清空 templates
- `on_after_apply`：收集对象 vec 中的模板定义（`:=` 语法），Root 模板对象以通过 GC 存活。重载时对已有项重新应用模板。根据 flow 方向设置 `vec_index`

### `begin` — 开始绘制
- 丢弃 `layout.align` 的主轴分量（防止非零对齐在列表未填满时产生多余的头部空白）
- 设置 `align` 为默认值，只为内部项传递交叉轴对齐
- 清空 `draw_align_list`

### `end` — 结束绘制（核心布局算法）
- 不重置 `at_end`（保留上次值，防止闪烁）
- 从 `draw_align_list` 收集所有已绘制项的位置和尺寸
- 排序后遍历计算 `first_id` 和 `first_scroll` 的精确值
- 分为两种情况处理：
  1. **视口顶部在列表顶部**（`list[0].index == range_start`）：计算 `first_pos`，限制回弹不超过 `max_pull_down`
  2. **视口不在顶部**：根据最后一项的位置偏移计算新的 `first_scroll`，确定是否有空间向下滚动
- 将测量的项高度记录到 `height_cache` 和 `height_tree`
- 更新未测量项的默认高度（使用新的平均值）
- **尾部跟踪**：`tail_range && !at_end` 时计算需要滚动的距离
  - `smooth_tail` 启用时启动弹簧阻尼动画（`Tailing` 状态）
  - 否则立即跳跃滚动
- 绘制滚动条（基于 `height_tree.total()` 或旧的分页估算）
- 更新滚动条位置
- **保留可见项**：非选中状态下清除不可见项（可选回收到 `reusable_items` 池）

### `next_visible_item` — 获取下一个可见项
- 实现两遍扫描绘制逻辑：
  1. **Down 阶段**：从 `first_id` 开始正向遍历，直到项超出视口底部
  2. **Up 阶段**：从 `first_id - 1` 开始反向遍历，直到项超出视口顶部或到达 `range_start`
  3. **DownAgain 阶段**：如果第一遍扫描后顶部仍有空白，从最后一项继续正向扫描
- 每步创建一个只传递交叉轴对齐的 Layout，主轴设为 0 对齐
- 返回当前项的 ID，供调用者绘制其内容

### `item` / `item_with_existed` — 获取或创建列表项
- 根据 `entry_id` 在 `ComponentMap` 中查找，存在且模板匹配则直接返回
- 模板不匹配或不存在时，优先从 `reusable_items` 池取用（需重置为模板默认值，防止状态泄漏）
- 池为空时通过 `WidgetRef::script_from_value` 新建
- 插入到 widget tree 中作为当前 PortalList 的子节点

### `set_item_range` — 设置数据范围
- 记录新的 `range_start`/`range_end`，初始化或调整 `height_tree` 大小
- 尾部跟踪模式下自动将 `first_id` 设到最后一项

### `update_scroll_bar` — 更新滚动条位置
- 使用 `height_tree` 计算基于像素的精确滚动位置：`prefix_sum(first_idx - 1) - first_scroll`
- 作为备选，使用旧的分页整数计算

### `delta_top_scroll` — 增量滚动
- 对 `first_scroll` 应用增量，在到达列表顶部时限制回弹范围
- 可选择过渡到 `Pulldown` 状态，或剪辑顶部溢出

### `smooth_scroll_to` — 平滑滚动到指定项
- 检查目标项是否已经在视口内（通过 `item_top_from_height_tree`），已可见则立即触发 `SmoothScrollReached`
- 计算滚动方向和初始偏移量，设置 `ScrollingTo` 状态
- 支持 `max_items_to_show` 参数限制跳跃跨度

### 跨项文本选择（Cross-boundary text selection）
- `hit_test_selection`：根据绝对坐标命中测试具体项和字符索引。处理视口上方/下方、项间空白等边界情况
- `get_selected_text`：收集选中范围内所有项的文本，拼接为字符串
- `update_item_selections`：对范围内的项设置选区，范围外的清除选中
- `select_all_visible`：选中所有可见项
- `clear_selection`：清除选中状态，隐藏剪贴板操作按钮

### `handle_event`（Widget）— 事件处理（核心逻辑）
- **滚轮滚动**（`FingerScroll`）：禁用尾部跟踪，应用滚动增量，触发 Scroll action
- **键盘导航**：Home/End/PageUp/PageDown/ArrowUp/ArrowDown，支持 Ctrl+A 全选
- **手指按下**（`FingerDown`）：记录 `was_scrolling` 状态，决定是否抑制子控件事件。进入拖拽模式，采集速度样本
- **手指移动**（`FingerMove`）：拖拽提交（超过阈值后）应用滚动增量。选中模式下更新光标位置
- **手指抬起**（`FingerUp`）：计算最后几个样本的速度，判断进入 Flick、Pulldown 或停止
- **选中模式**：在处理 `TextCopy`/`TextCut` 时提供选中文本

### 滚动状态处理
- `ScrollingTo`：每帧检查目标项是否到达目标位置（通过 `height_tree`），到达后触发 `SmoothScrollReached`
- `Flick`：每帧按 `flick_scroll_decay` 衰减速度，低于 `flick_scroll_minimum` 时停止
- `Pulldown`：每帧将 `first_scroll` 乘以 0.85，低于 1px 时归零
- `Tailing`：弹簧阻尼模型，`velocity = (velocity + spring_force) * damping`，优雅吸收内容快速追加

### `draw_walk`（Widget）
- 使用 `DrawStateWrap` 两阶段绘制：先 `begin`，然后在下一绘制步骤中 `end`

### PortalListRef 方法
- 提供 `set_first_id`、`set_tail_range`、`is_at_end`、`scrolled`、`smooth_scroll_to`、`scroll_to_end` 等常用 API
- `debug_scroll_state_line`：返回当前滚动状态的调试字符串
