# text_input/tests/ui.rs

TextInput 组件的 UI 集成测试和测试框架本身的功能验证。

## 测试用例

### `fill_clear_and_submit_singleline_input`
1. **第6-12行**：输入 "hello" → 验证 → 清除 → 验证为空 → 再次输入 → 验证
2. **第14行**：模拟回车提交
3. **第15-16行**：验证状态标签更新
4. **第17行**：验证控制台日志包含预期内容

### `waits_for_visible_inputs`
- **第20-22行**：简单验证 email 输入框可见

### `missing_selector_reports_useful_error`
- **第25-32行**：点击不存在的 selector 验证返回 `TestError`，错误信息包含 "matched no visible widgets"

### `type_selector_reports_multiple_matches`
- **第36-43行**：使用 `widget_type("TextInput")` 选择器时会匹配多个 widget，验证错误信息包含 "matched multiple widgets" 和 "input_singleline"

### `captures_failure_artifacts`
- **第47-71行**：单元测试验证测试框架的失败产物捕获机制：
  1. 手动触发测试失败
  2. 验证生成了 `failure.txt`, `logs.txt`, `widget-tree.txt`, `widget-snapshot.json`, `failure-screenshot.png`

## 关键 API

- `run_with_config(config, |app| ...)` — 手动运行测试配置
- `TestConfig::current_package(...)` — 测试配置生成
- `TestError::new(...)` — 自定义测试错误
