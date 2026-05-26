# pdf_view.rs — PDF 文档查看器

## 整体职能
`PdfView` widget 在 Makepad 应用中嵌入 PDF 文档渲染能力。它基于 `pdf` crate（底层调用 `pdf.rs` / `lopdf`）将 PDF 页面解析为位图或矢量图形，并支持缩放、滚动和页面导航。

## 主要数据结构
- **`PdfView`**：顶层 widget，包含 `draw_bg`（背景）、`draw_page`（页面内容绘制）、`scroll`（滚动状态）、`zoom`（缩放级别）、`current_page`（当前页码）、`total_pages`（总页数）等字段。
- **`PdfDocument`**：封装 `pdf::Document`，管理 PDF 文件的加载、页面解析和渲染缓存。
- **`PdfPage`**：表示单个渲染后的页面，存储位图数据（`Vec<u8>` 格式的 RGBA 像素）或矢量绘制命令列表。
- **`PdfPageCache`**：LRU 缓存，管理已渲染页面的内存，避免重复渲染。

## 方法与实现逻辑

### `fn script_component` — 脚本注册
注册 `PdfView` widget 到脚本运行时。通过 `set_type_default` 设置默认背景色（白色）、初始缩放级别 1.0 和滚动条样式。

### `fn load_document` — 加载 PDF 文档
接收文件路径 `&str` 或字节数据 `&[u8]`。调用 `pdf::Document::load()` 解析文档结构。获取页面数量 `num_pages` 并初始化 `PdfPageCache`。如果加载失败，将错误信息存入 `status_text`。

### `fn draw_walk` — 页面绘制
1. 调用 `draw_bg.draw_abs(cx, rect)` 绘制白色背景。
2. 根据 `zoom` 计算页面的目标绘制尺寸。
3. 检查 `PdfPageCache` 中是否有当前页的缓存；如果没有，触发后台渲染任务。
4. 如果有缓存，调用 `draw_page.draw_abs(cx, page_rect)` 将渲染后的页面绘制到屏幕上。
5. 应用 `scroll` 偏移实现页面滚动。支持连续滚动模式（所有页面连续排列）和单页模式。

### `fn handle_event` — 事件处理
- **滚轮事件**：缩放（Ctrl+滚轮）或上下滚动页面。
- **键盘事件**：PageUp / PageDown / 方向键切换页面。
- **触摸手势**：通过内部的 `TouchGesture` 处理触摸拖动和缩放手势。
- **鼠标拖动**：在页面区域拖拽时平移视图。

### `fn render_page` — 页面渲染（异步）
在后台线程中调用 `pdf::Page::render()` 将 PDF 页面渲染为 RGBA 位图。渲染完成后通过消息通道将结果传回主线程，更新 `PdfPageCache`。

### `fn set_zoom` — 设置缩放级别
更新 `zoom` 值（范围 0.25 到 5.0），清除页面渲染缓存，触发重绘。支持通过鼠标滚轮或捏合手势调节。

### `fn go_to_page` — 页面导航
设置 `current_page` 为目标页码（范围检查）。如果目标页面未缓存，触发后台渲染。可选支持平滑翻页动画。

### `fn get_table_of_contents` — 获取目录
解析 PDF 文档的书签结构，返回 `Vec<PdfBookmark>`，包含层级、标题和目标页码。供 UI 的目录面板使用。

### `fn search_text` — 文本搜索
在当前页面或全文中搜索指定文本，返回匹配的位置和页码列表。高亮显示所有匹配区域。
