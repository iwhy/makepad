# nav_control.rs — 键盘 Tab 导航控制

## 整体职能
`NavControl` widget 提供基于 **Tab 键的焦点导航** 管理，实现键盘可访问性（keyboard accessibility）。它维护一个焦点循环队列，允许用户在可交互 widget 之间按 Tab/Shift+Tab 顺序切换焦点，以及 Enter/Escape 激活或取消。

## 主要数据结构
- **`NavControl`**：顶层 widget，包含 `focusable_items`（可聚焦项列表）、`focused_index`（当前聚焦索引）、`focus_ring`（聚焦环绘制句柄）、`enabled`（启用标记）等字段。
- **`NavItem`**：表示单个可聚焦项，包含 `widget_id`（关联的 `WidgetId`）、`label`（文本标签）、`enabled`（是否可聚焦）、`group_id`（可选分组 ID）。
- **`NavDirection`**：枚举，定义导航方向 `Forward` / `Backward`。
- **`NavControlAction`**：Action 枚举，当导航发生时冒泡的事件，包括 `ItemFocused(id, index)`、`ItemActivated(id, index)`、`NavCompleted` 等。

## 方法与实现逻辑

### `fn script_component` — 脚本注册
注册 `NavControl` 及关联的 `NavItem`、`NavDirection`、`NavControlAction` 到脚本运行时。定义默认的 `focus_ring` 样式（2px 蓝色边框）。

### `fn draw_walk` — 绘制与焦点环
1. 遍历 `focusable_items`，为每个 `NavItem` 分配绘制区域。
2. 如果某个 `NavItem` 当前处于聚焦状态（`index == focused_index`），调用 `focus_ring.draw_abs(cx, item_rect)` 在其周围绘制聚焦环（带虚线或实线边框）。
3. 返回绘制步骤让子 widget 继续渲染。

### `fn handle_event` — 键盘事件处理
监听 `KeyEvent`：
- **Tab**（无 Shift）：`focus_next(NavDirection::Forward)`，将焦点移到下一个可聚焦项。
- **Shift+Tab**：`focus_next(NavDirection::Backward)`，移到上一个。
- **Enter / Space**：`activate_current()`，触发当前聚焦项的 `ItemActivated` Action。
- **Escape**：`clear_focus()`，取消所有聚焦状态。
防止事件冒泡（`cx.consume_event()`）以避免父容器同时触发导航。

### `fn focus_next` — 移动到下一/上一项
根据 direction 计算下一个索引（循环或线性）。跳过 `enabled = false` 的项。更新 `focused_index` 并触发 `ItemFocused` Action。

### `fn focus_by_id` — 按 ID 聚焦
根据 `WidgetId` 查找对应的 `NavItem` 并设置焦点。如果目标不在列表中则返回 false。

### `fn register_item` — 注册可聚焦项
允许子 widget 在初始化时将自身注册为可聚焦项。注册时自动分配一个 `NavItem` 结构体并插入排序的 `focusable_items` 列表。

### `fn remove_item` — 移除聚焦项
当 widget 卸载时，从焦点列表中移除对应项。如果当前聚焦项被移除，将焦点移到下一个可用项。
