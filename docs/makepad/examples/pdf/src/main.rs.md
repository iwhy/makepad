# pdf/src/main.rs

演示 Makepad 的 `PdfView` 组件，实现一个 PDF 文件查看器。

## 整体结构

- **第7-24行**：`script_mod!` UI 定义 — 使用 `PdfView` 作为主要组件，Fill/Fill 填满窗口
- **第26-58行**：`try_load_pdf` — 延迟加载 PDF 数据：
  1. 使用多种路径方式查找 PdfView widget（`direct`, `path_main`, `path_body`, `flood`）
  2. 找到后调用 `load_pdf_data` 加载数据
- **第60-66行**：App 结构体，`pdf_data: Option<Vec<u8>>` 存储 PDF 字节数据
- **第68-83行**：`MatchEvent`：
  - `handle_startup`：解析命令行参数查找 `--pdf` 或 `--file` 指定的文件路径；若未提供则调用 `generate_demo_pdf()` 生成演示 PDF
  - `handle_actions`：空实现
- **第85-98行**：`AppMain` — `handle_event` 在 Draw 事件触发时调用 `try_load_pdf`（确保在 widget 树就绪后才加载）
- **第100-120行**：`find_pdf_path_arg` — 命令行参数解析函数
- **第122-124行**：`generate_demo_pdf` — 生成测试用 PDF（使用 `makepad_pdf_parse` crate）

## 关键 API

- `PdfView` — PDF 渲染 widget
- `PdfView::load_pdf_data(cx, data)` — 加载 PDF 字节数据
- `self.ui.widget_flood(cx, ids!(...))` — 通过泛洪查找跨组件边界的 widget
