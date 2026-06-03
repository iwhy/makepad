# todo/src/main.rs

演示完整的 Todo 列表应用，使用 `PortalList` 实现虚拟列表渲染、全局状态管理和自定义 SVG 图标。

## 整体结构

### 数据层（第8-23行）

- `TodoItemData` 结构体：包含 `text`, `tag`, `done` 三个字段
- `TODOS` 全局静态变量：`LazyLock<RwLock<Vec<TodoItemData>>>` 线程安全全局状态

### UI 定义（第25-265行）

- **第29-48行**：4 个 SVG 图标（`IconCheck`, `IconTrash`, `IconRocket`, `IconClipboard`）使用 `Vector` 组件+Path 绘制
- **第50-106行**：`TodoRow` — 每条待办项的模板，包含 CheckBox、Label、tag RoundedView、删除 ButtonFlatter
- **第108-115行**：`EmptyState` — 列表为空时显示的占位 UI
- **第117-130行**：`TodoList` — 注册为 widget 的 PortalList，模板包含 `Item` 和 `Empty`

### Rust 侧（第274-316行）

`TodoList` widget 的 `draw_walk` 实现：
- 从全局 `TODOS` 读取数据
- 空列表时使用 `Empty` 模板显示空状态
- 非空时逐项渲染，设置 checkbox 状态、标签文本、tag 文本

### 应用逻辑（第319-412行）

- `add_todo`：添加新待办，清空输入框
- `clear_done`：删除所有已完成项
- `toggle_item`：切换待办完成状态
- `delete_item`：删除指定待办
- `sync_status`：更新底部状态栏（"X remaining / Y total"）

事件处理（`handle_actions`）：
- 输入框回车 → 添加待办
- 添加按钮点击 → 添加待办
- clear_done 按钮 → 清除已完成的
- PortalList 中每项的 checkbox 变更 → toggle
- PortalList 中每项的删除按钮 → delete

## 关键 API

- `list.item(cx, item_id, id!(Item))` — 获取 PortalList 中的模板实例
- `item.check_box(cx, ids!(check))` — 在模板实例内部查找组件
- `list.items_with_actions(actions)` — 迭代触发事件的项目
- `CachedView` — 模板缓存包装
- `Animate::No` / `Animate::Yes` — 动画控制
