# `page_flip.rs` — 页面切换器

## 作用
实现一个多页面切换控件，同时只显示一个活跃页面。支持懒加载（按需创建页面）和预创建所有页面两种模式。

## 关键结构

### `PageFlip`
| 字段 | 类型 | 说明 |
|------|------|------|
| `active_page` | `LiveId` | 当前活跃页面 ID |
| `lazy_init` | `bool` | 懒加载模式 |
| `templates` | `ComponentMap<LiveId, ScriptObjectRef>` | 页面模板 |
| `pages` | `ComponentMap<LiveId, WidgetRef>` | 已实例化的页面 |

## 方法详解

### `ScriptHook` 实现
- `on_before_apply`：重载时清空 templates
- `on_after_apply`：收集 vec 中的页面模板。如果已有对应页面实例，重载时对其应用更新。非懒加载模式下立即创建所有页面

### `page`
- 根据 `page_id` 获取页面实例。不存在时创建（懒加载的核心），插入到 widget tree 中
- 区分 `on_after_apply` 中的 `script_from_value_scoped`（有 scope）和 `page()` 中的 `script_from_value`（无 scope）

### `begin` / `end`
- 标准的海龟布局块

### `WidgetNode` 实现
- `children`：遍历所有页面，提供给 widget tree 遍历用
- `walk` / `area` / `redraw`：标准实现

### `handle_event`（Widget）
- 对需要可见性的事件，只转发给 `active_page`；其他事件（如定时器）转发给所有页面
- 使用 `cx.group_widget_actions` 分组 action

### `draw_walk`（Widget）
- 调用 `self.page(cx, active_page)` 获取或创建当前页
- 使用 `draw_state.begin_with` 预热 walk 值
- 只绘制当前活跃页面

### `set_active_page`
- 切换活跃页面，必要时创建该页面
- 仅当 ID 变化时才触发重绘
