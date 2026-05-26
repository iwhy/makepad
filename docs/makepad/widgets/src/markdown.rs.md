# markdown.rs — Markdown 渲染组件

## 整体职能
`Markdown` widget 是一个支持內嵌图片、代码块高亮、标题锚点链接和主题定制的完整 Markdown 渲染引擎。它基于 `Html` widget 的能力，将 Markdown 文本解析为内部的文档树并逐节点绘制。

## 主要数据结构
- **`Markdown`**：顶层 widget 结构体，持有 `html: Html` 子 widget、滚动位置 `scroll_pos`、加载状态 `loading_pending`、以及可选的依赖映射 `dep_map`。通过 `#[live]` 暴露 `font_size`、`text_style` 等排版属性。
- **`MarkdownDep`**：用于描述资源之间依赖关系的最小单位，包含 `from` 和 `to` 两个 `LiveId`。
- **`ScriptLiveIdMap` / `ScriptLiveIdSet`**：对 `HashMap<LiveId, LiveId>` / `HashSet<LiveId>` 的 newtype 封装。

## 方法与实现逻辑

### `fn script_component` — 脚本组件注册
以 `#[derive(Script, ScriptHook)]` 为基础，调用 `Markdown::script_component(vm)` 将 `mod.widgets.MarkdownBase` 注册到脚本运行时。随后通过 `set_type_default()` 定义带默认属性的 `mod.widgets.Markdown`。

### `fn set_text` — 设置 Markdown 源文本
接收 `&str`，将其存入 `self.text` 并通过 `self.parse()` 立即触发解析。`text` 字段同时以 `#[live]` 暴露给脚本，支持动态修改。

### `fn handle_event_with_dep_map` — 带依赖映射的事件处理
接受额外的 `&[MarkdownDep]` 参数。在 `MatchEvent::handle_actions` 基础上，先检查 `loading_pending` 标记，再处理 `Html` widget 触发的各种 Action。收到 `HtmlAction::Loaded` 时，根据 dep_map 遍历所有已加载的资源文件并处理链接跳转。

### `impl Widget for Markdown` — Widget trait 实现
- **`draw_walk`**：将 `font_size`、`text_style` 等属性透传给内部的 `html` widget，然后调用 `html.draw_walk(cx, scope, walk)` 进行实际的递归绘制。
- **`handle_event`**：调用 `handle_event_with_dep_map`，传入空 dep_map。此方法将接收到的事件（鼠标、键盘、滚动）透传给 `html` widget，并根据 `HtmlAction` 做出响应，例如点击链接时通过 `cx.open_url()` 在系统浏览器中打开。

### `fn handle_markdown_action` — 链接点击处理
根据 `HtmlAction::LinkClick` 中的 URL 类型进行分发：`http://` / `https://` 用 `cx.open_url` 打开；`#` 开头的锚点调用 `cx.open_link(href, false)` 实现页面内跳转。

### `fn reload` — 重新加载
清空当前文本，然后调用 `set_text` 重新设置，触发一次完整的解析-渲染循环。
