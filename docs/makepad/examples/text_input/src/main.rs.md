# text_input/src/main.rs

演示 TextInput 组件在多种 IME 配置模式下的使用。

## UI 定义（第7-148行）

按顺序展示 9 种 TextInput 配置：

| 编号 | 模式 | 关键配置 | 行号 |
|------|------|---------|------|
| 1 | 默认多行 | `is_multiline: true` | 29-33 |
| 2 | 单行 | `return_key_type: Done` | 40-44 |
| 3 | 邮件 | `input_mode: Email`, `autocorrect: Disabled` | 51-57 |
| 4 | URL | `input_mode: Url`, `autocapitalize: None` | 64-71 |
| 5 | 数字 | `input_mode: Decimal` | 79-83 |
| 6 | 搜索 | `input_mode: Search`, `return_key_type: Search` | 90-95 |
| 7 | 密码 | `is_password: true` | 102-108 |
| 8 | 全大写 | `autocapitalize: AllCharacters` | 115-119 |
| 9 | ASCII | `input_mode: Ascii`, `autocorrect: Enabled` | 125-131 |

## Rust 事件处理（第156-176行）

`handle_actions` 逐一检查 7 个输入框的 `returned` 事件，将返回值显示在 `status_label` 上。

## 关键 API

- `TextInput::returned(actions)` — 检测回车/提交事件，返回 `(String, Modifiers)`
- `input_mode` 枚举：`Email`, `Url`, `Decimal`, `Search`, `Ascii`
- `autocorrect`：`Enabled` / `Disabled`
- `autocapitalize`：`Sentences`, `Words`, `AllCharacters`, `None`
- `is_password`：密码模式（文字隐藏）
- `is_multiline`：多行模式
- `return_key_type`：返回按钮类型（`Done`, `Go`, `Search` 等）
