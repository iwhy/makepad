# `desktop_file_tree.rs` — Studio 文件树组件

## 文件作用

实现文件树的两种显示模式：标准树状视图（`FileTree`）和过滤列表视图（`PortalList`）。通过 `PageFlip` 在两种视图间切换。

## script_mod 定义

### 常量
- `STUDIO_FILE_TREE_ROW_HEIGHT = 28.0` — 过滤列表行高
- `STUDIO_FILE_TREE_NODE_HEIGHT = 22.0` — 树状视图节点高

### 子组件

**`FilteredFileItem`**: 过滤结果列表项（View）：
- 交替背景色（`is_even` 实例变量控制）
- 圆形 Git 状态指示点（`status_dot`，使用 SDF 圆形着色）
- `row_button`（全宽按钮，透明背景，显示路径文本）

**`FilteredFileEmpty`**: 空列表占位项。

**`DesktopFileTree`**: 完整文件树组件：
```
DesktopFileTree (View)
 └── page_flip (PageFlip)
     ├── file_tree_page (FileTree) — 标准树状视图
     └── filter_list_page (PortalList) — 过滤结果
         ├── Item := FilteredFileItem
         └── Empty := FilteredFileEmpty
```

## 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct DesktopFileTree {
    #[deref] view: View,
    #[rust] filter_active: bool,
}
```

### Action 枚举

```rust
pub enum DesktopFileTreeAction {
    FileClicked(LiveId),        // 文件节点点击
    FolderClicked(LiveId),      // 文件夹节点点击
    FilteredPathClicked(String), // 过滤结果路径点击
    None,
}
```

## Widget trait 实现

### `draw_walk`
1. 从 scope data 检查当前 mount 是否有活跃的过滤条件
2. 如果 `filter_active` 状态变化，切换 `page_flip` 的活动页面
3. 委托 `view.draw_walk` 进行绘制
4. 在 step 回调中：
   - 如果 step 是 `FileTree`（`filter_active = false`），调用 `data.file_tree.draw(cx, file_tree)`
   - 如果 step 是 `PortalList`（`filter_active = true`），调用 `draw_filtered_list`

### `draw_filtered_list`
1. 计算填充视口的空行数
2. 如果过滤结果为空，显示 "Searching..." 或 "No matches"
3. 对每个可见项：
   - 设置交替背景色
   - 设置 Git 状态点颜色（通过 `script_apply_eval!` 更新 `status_dot` 实例变量）
   - 设置按钮文本为路径，附加 `FilteredFileRowData::Path` 作为 action 数据

### `handle_event`
处理两种事件源：
1. `FileTree` 的 `FileClicked` / `FolderClicked` → 转发为 `DesktopFileTreeAction`
2. `PortalList` 中 item 的 `row_button` 点击 → 读取 `FilteredFileRowData` → 转发为 `FilteredPathClicked`

## DesktopFileTreeRef 方法

- `file_clicked` / `folder_clicked` — 从 actions 中提取文件/文件夹点击
- `filtered_path_clicked` — 从 actions 中提取过滤路径点击
- `set_folder_is_open` — 展开/折叠文件树文件夹

## 与 AppData 的关系

`DesktopFileTree` 通过 scope data 获取 `AppData`，从中读取：
- `data.active_mount` → 确定当前 mount
- `data.file_tree` → `FlatFileTree` 用于树状视图绘制
- `mount_state.file_filter` / `file_filter_results` / `file_filter_pending` → 过滤状态和结果

**不直接与 hub 通信**——过滤查询由 `app_backend.rs` 中的 `set_mount_file_filter` 管理。
