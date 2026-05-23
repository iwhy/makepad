# `flat_list.rs` — 扁平列表

## 作用
实现一个简单的扁平列表容器，所有子项同时渲染（非虚拟化）。适用于项数量较少、不需要虚拟优化的场景。

## 关键结构

### `WidgetItem`
| 字段 | 类型 | 说明 |
|------|------|------|
| `widget` | `WidgetRef` | 小部件实例 |
| `template` | `LiveId` | 模板标识符 |

此结构也被 `portal_list.rs` 复用。

### `FlatList`
| 字段 | 类型 | 说明 |
|------|------|------|
| `scroll_bars` | `ScrollBars`（`#[redraw]`） | 滚动条 |
| `templates` | `HashMap<LiveId, ScriptObjectRef>` | 模板映射 |
| `items` | `ComponentMap<LiveId, WidgetItem>` | 子项集合 |

## 方法详解

### `ScriptHook` 实现
- `on_before_apply`：重载时清空 templates
- `on_after_apply`：收集 vec 中的模板（与 PortalList 相同的模式）。重载时对已有项重新应用。根据 flow 方向设置 `vec_index`

### `begin` / `end`
- 分别调用 `scroll_bars.begin` 和 `scroll_bars.end`，委托滚动条管理

### `space_left`
- 计算海龟布局中剩余的高度空间：`rect.size.y - used.y`

### `item`
- 根据 `LiveId` 在 `ComponentMap` 中查找项。存在则返回，不存在则从模板新建并插入
- 与 PortalList 不同，FlatList 不做虚拟化，所有项同时存在

### `handle_event`（Widget）
- 将事件转发给 `scroll_bars` 和所有子项
- 使用 `cx.group_widget_actions` 将子项 action 分组到列表 uid 下

### `draw_walk`（Widget）
- 使用 `DrawStateWrap` 两阶段绘制

### `FlatListRef` 方法
- `item`：通过 `borrow_mut` 获取或创建项
- `items_with_actions`：遍历所有 action，找到本列表中匹配的子项 action，返回 `(LiveId, WidgetRef)` 列表

### `FlatListSet` 方法
- `items_with_actions`：聚合多个 FlatList 的 action 匹配结果
