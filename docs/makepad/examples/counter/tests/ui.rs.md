# counter/tests/ui.rs

Makepad UI 集成测试示例，使用 `makepad_test` 框架。

## 测试用例

### `counter_smoke`
1. **第4-10行**：通过 `Selector::id("increment_button")` 定位按钮，`wait_visible()` 等待可见后 `click()`；然后检查 `counter_label` 的文本是否为 `"Count: 1"`

### `counter_tracks_multiple_clicks`
1. **第12-18行**：连续点击 3 次按钮，验证计数器文本变为 `"Count: 3"`

## 关键 API

- `Selector::id(...)` — 按 widget ID 查找
- `wait_visible()` — 等待 widget 可见
- `click()` — 点击 widget
- `wait_text(...)` — 等待 widget 文本内容匹配
