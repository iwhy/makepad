# `dock.rs` — Dock 面板系统

## 作用
Makepad 最复杂的高级布局控件——实现可拖拽的标签页式面板系统，支持：
- 标签页管理（打开/关闭/切换）
- 拖拽重排标签页
- 面板拆分（Splitter 根节点）
- 标签动画（展开/收起的平滑动画）
- 右键上下文菜单

## 自定义 Shader

### `DrawDockSlot`
- 标签页标签绘制，支持 `hover`、`active`、`top_inset` 等实例参数

## 关键结构

### `DockAction`
| 变体 | 说明 |
|------|------|
| `Open(LiveId)` | 打开标签页（从树节点） |
| `Close(LiveId)` | 关闭标签页 |
| `Detach(LiveId)` | 分离标签页（未来实现） |
| `Select(LiveId)` | 切换选中标签页 |
| `ContextMenu` | 打开右键菜单 |

### `Dock`
| 字段 | 类型 | 说明 |
|------|------|------|
| `view` | `View`（`#[deref]`） | 内部 View |
| `draw_tab_bar` | `DrawQuad` | 标签栏背景 |
| `draw_overflow` | `DrawQuad` | 溢出指示器 |
| `dock_splitter` | `DockSplitter` | 分隔条配置 |
| `dock_tabs` | `DockTabs` | 标签页配置 |
| `dock_stack` | `Option<DockStack>` | 标签页堆栈管理 |
| `dock_contents` | `Option<AndThen<f64, true>>` | 内容绘制区域 |

### `DockStack`
- 内部管理 `DockTree`（树形布局结构），`ActiveTabs`（映射）和选中状态
- 标签按添加顺序排列，支持 `select`、`select_next`、`close_active_tab`、`select_nth` 操作

### `DockTree`
- `Split`（水平/垂直分割，包含 `ratio` 和两个子树）
- `Leaf`（包含标签和内容区域）

### `DockSplitter`
| 字段 | 类型 | 说明 |
|------|------|------|
| `handle_tree` | `HandleTree` | 拖拽分隔条 |
| `content_a` / `content_b` | `WidgetRef` | 分割面板 |
| `split_type` | `SplitType` | 水平/垂直 |

### `DockTabs`
| 字段 | 类型 | 说明 |
|------|------|------|
| `tab_header` | `DockTabsHeader` | 标签页眉 |
| `tab_content` | `DockTabContent` | 标签内容区域 |

## 方法详解

### `begin` / `end`
- 清空/结束 `dock_stack` 和 `dock_splitter` 状态
- `end` 中计算标签展开/收起动画，管理内容绘制区域

### `tab_header` 绘制
- 按标签顺序绘制，超出宽度时显示溢出指示器
- 活跃标签有高亮底部条（`top_inset`）

### `ContentAnimation` 标签展开/收起动画
- `is_animating`：检查是否有动画正执行
- 计算标签的 `target_bounds`（目标边界）并通过 `dock_stack` 映射
- 使用 `AndThen` 嵌套布局构建内容动画——一个标签展开后跟一个标签

### `handle_event`（Widget）
- 处理标签点击（切换选中标签）
- 处理标签关闭按钮
- 处理标签拖拽重排（检测长按后启动）
- 处理右键上下文菜单
- 处理 DockTree 面板拖拽分隔条
- 处理通过 `NewAction` 打开新标签（如从 FileTree 点击文件）

### `open_in_tab`
- 查找或创建标签页，设置选中

### `close_tab`
- 关闭标签，自动选中相邻标签

### `DockRef` 方法
- `select`：切换到指定标签
- `open_in_tab`：通过节点 ID 打开/切换标签
- `new_tab`：添加新标签（指定 ID、内容模板、标签文本）
- `tab_selected` / `tab_closed` / `context_menu`：检查 dock actions
- `get_dock_tree`：获取当前布局结构
