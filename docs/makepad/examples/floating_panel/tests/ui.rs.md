# floating_panel/tests/ui.rs

浮动面板示例的 UI 集成测试。

## 测试用例

### `panel_input_updates_main_window_status`
1. **第5-8行**：在 `panel_window` 中找到输入框，输入 "hello"，验证值为 "hello"
2. **第9-10行**：验证主窗口 `status_label` 更新为 "Panel typing: \"hello\""
3. **第11行**：模拟回车键
4. **第12-13行**：验证主窗口状态更新为 "Panel submitted: \"hello\""
5. **第14-17行**：点击面板中的 ping_button，验证主窗口状态更新

### `floating_panel_can_be_dragged`
1. **第21-23行**：在面板窗口中找到 `drag_target` 并执行拖拽操作（140px 水平，40px 垂直）
2. **第24-26行**：验证拖拽距离显示为 "Drag delta: 140, 40"

## 关键 API

- `Selector::id(...).window("panel_window")` — 限定到特定窗口的选择器
- `fill(...)` — 在输入框中输入文本
- `wait_value(...)` — 等待输入框的值
- `press_return()` — 模拟回车键
- `drag_by(x, y)` — 模拟拖拽
