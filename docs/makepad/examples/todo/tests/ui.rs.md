# todo/tests/ui.rs

Todo 应用的完整 UI 集成测试。

## 测试用例

### `todo_smoke`
1. **第5-6行**：验证 todo 输入框可见
2. **第7-8行**：验证初始待办文本 "Get AI to control UI" 可见

### `todo_add_toggle_delete_and_disambiguate_duplicates`
1. **第12-14行**：取消第一个 checkbox 的选中
2. **第16-21行**：添加 "Write tests" 两次（重复文本）
3. **第23-24行**：验证有 2 个 "Write tests"
4. **第25-28行**：验证第 2 个 checkbox 可点击选中
5. **第29-31行**：删除第 3 个 "x" 按钮，验证只剩 1 个 "Write tests"

### `todo_clear_completed_removes_only_checked_items`
1. **第37-49行**：添加三个新待办
2. **第51-56行**：勾选第 2 个和第 4 个 checkbox
3. **第58行**：点击 "Clear completed"
4. **第60-68行**：验证只有未勾选的 "Email Ada" 保留（`wait_count(1)`），其余均被删除

## 关键 API

- `Selector::widget_type("CheckBox").nth(N)` — 按类型和索引定位
- `wait_count(N)` — 验证匹配元素数量
- `fill("text").wait_value("text")` — 输入文本并立即验证
