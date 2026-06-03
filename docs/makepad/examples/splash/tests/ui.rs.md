# splash/tests/ui.rs

Splash 示例的 UI 集成测试。

## 测试用例

### `splash_modal_smoke`
1. **第5-6行**：按 widget 类型和文本查找 "Modal" 标签页并点击
2. **第8-10行**：点击 "Open Modal" 按钮
3. **第11-12行**：验证状态文本变为 "Modal status: Basic Modal Open"
4. **第13-15行**：点击 "Close Modal" 按钮
5. **第16-17行**：验证状态文本变为 "Modal status: Closed via button"

### `splash_toggle_and_dropdown_smoke`
1. **第21-24行**：切换到 "Toggles" 标签页
2. **第25-28行**：点击 checkbox，验证其变为选中状态
3. **第29-32行**：点击 toggle，验证其变为选中状态
4. **第34-36行**：验证 dropdown 显示 "Option A"

### `splash_media_scroll_smoke`
1. **第40-43行**：切换到 "Media" 标签页
2. **第44-46行**：定位测试图片，执行滚动操作
3. **第47-48行**：验证滚动后 "Loading Spinner" 标签可见

## 关键 API

- `Selector::widget_type("DockTab").text_exact("Modal")` — 多条件组合选择器
- `scroll(0.0, -1200.0)` — 模拟滚动操作
- `wait_checked(true)` — 等待勾选状态
