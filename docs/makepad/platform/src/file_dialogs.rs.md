# `file_dialogs.rs` — 文件选择对话框

## 概述

实现构建器模式的跨平台文件打开/保存对话框接口，基于简化版的 `native_dialog_rs` 接口。该模块定义对话框的配置数据结构，实际的平台原生对话框显示由各平台后端（`cx.rs` 中的 `CxFileDialog` 分支）实现。

## 核心类型

### `struct Filter`
文件类型过滤器，包含：
- **`description: String`**：显示给用户的描述文本（如"PNG 图像文件"）。
- **`extensions: Vec<String>`**：对应的扩展名列表（如 `["png"]`）。

### `struct FileDialog`
构建器模式的对话框配置结构，包含：
- **`filename: Option<String>`**：默认文件名。在 macOS 和 zenity 的打开对话框中为 no-op（无此输入字段）。
- **`location: Option<PathBuf>`**：初始目录路径。未设置时使用当前工作目录。
- **`filters: Vec<Filter>`**：文件类型过滤器列表。对"选择目录"类型的对话框为 no-op。
- **`title: Option<String>`**：对话框窗口标题。

## 方法

### `new() -> Self`
创建空白的 FileDialog 构建器，所有字段为 `None` 或空向量。

### `set_title(title) -> Self`
设置对话框窗口标题。返回 `self` 以支持链式调用。

### `set_filename(filename) -> Self`
设置默认文件名。在 macOS 和 zenity 的打开对话框中不生效。

### `reset_filename() -> Self`
清除默认文件名设置。

### `set_location(path) -> Self`
设置对话框初始显示的目录。

### `reset_location() -> Self`
清除目录设置，恢复为当前工作目录。

### `add_filter(description, extensions) -> Self`
添加文件类型过滤器。若 `extensions` 为空向量则直接 `panic`，因为每个过滤器至少需要一个扩展名。

### `remove_all_filters() -> Self`
清除所有已添加的文件过滤器。

`Default` 实现直接委托给 `new()`。
