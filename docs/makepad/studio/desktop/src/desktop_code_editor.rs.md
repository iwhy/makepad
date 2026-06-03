# `desktop_code_editor.rs` — Studio 代码编辑器包装

## 文件作用

一个轻量级的 `CodeEditor` 包装组件，从 `AppData` 中获取 `CodeSession` 进行绘制。本身不包含编辑器逻辑，完全委托给 `makepad_code_editor::CodeEditor`。

## script_mod 定义

```rust
mod.widgets.DesktopCodeEditorBase = #(DesktopCodeEditor::register_widget(vm))
mod.widgets.DesktopCodeEditor = set_type_default() do mod.widgets.DesktopCodeEditorBase {
    editor := CodeEditor {}
}
```

- 注册 `DesktopCodeEditor` widget
- 设置默认样式——仅包含一个 `CodeEditor` 子组件

## 结构体

```rust
#[derive(Script, ScriptHook, WidgetRef, WidgetSet, WidgetRegister)]
pub struct DesktopCodeEditor {
    #[uid] uid: WidgetUid,
    #[live] pub editor: CodeEditor,
}
```

实现了 `WidgetNode` trait，将所有方法委托给 `self.editor`。

## Widget trait 实现

### `draw_walk`
1. 从 widget tree 中提取 tab_id（路径倒数第二个元素）
2. 从 scope data 中获取 `AppData`
3. 如果 tab_id 对应的 session 存在，调用 `editor.draw_walk_editor` 绘制完整编辑器
4. 否则调用 `editor.draw_empty_editor` 绘制空白编辑器

**关键模式**: session 存储在 `AppData.sessions` 中，通过映射到 tab_id 来管理。编辑器本身不持有 session 状态。

### `handle_event`
1. 同样提取 tab_id
2. 从 `AppData.sessions` 获取 session
3. 委托 `editor.handle_event` 处理事件
4. 编辑器产生的 action 通过 `cx.widget_action` 转发

## DesktopCodeEditorRef 扩展

### `set_cursor_and_scroll`
通过 `borrow_mut` 获取内部可变引用，委托到 `CodeEditor::set_cursor_and_scroll` 并设置键盘焦点。用于日志跳转功能。

## 与 Hub 的交互

编辑器本身不直接与 hub 通信。文件读写通过 `App` 层面的 `ClientToHub::OpenTextFile` / `SaveTextFile` 进行，session 数据在 `app_messages.rs` 的 `apply_editor_text_update` 中从 hub 的响应更新。
