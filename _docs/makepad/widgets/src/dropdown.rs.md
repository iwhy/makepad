# `dropdown.rs` — 下拉选择器

## 作用
实现一个下拉选择器控件，包含以下子控件：
- 点击输入框触发弹出列表
- 弹出窗口居中对齐于输入框
- 支持滚动列表选择
- 可自定义每个选项的文本和值

## 脚本 DSL 结构

```
DropDown {
    input: TextInput (显示当前选中值)
    popup: View (弹出列表容器)
    list: PortalList (选项虚拟滚动列表)
    item: View.template (选项模板)
}
```

## 关键结构

### `DropDownAction`
| 变体 | 说明 |
|------|------|
| `Selected(u64)` | 选中了指定值 |
| `None` | 无操作 |

### `DropDown`
| 字段 | 类型 | 说明 |
|------|------|------|
| `draw_bg` | `DrawQuad` | 背景绘制 |
| `draw_caret` | `DrawQuad` | 下拉箭头图标 |
| `input` | `TextInput` | 文本输入框 |
| `popup` | `View`（弹出窗口） | 弹窗容器 |
| `list` | `PortalList` | 选项列表 |
| `option_template` | `ScriptObjectRef` | 选项模板 |
| `values` | `Vec<(u64, String)>` | (值, 文本) 选项列表 |
| `selected` | `Option<u64>` | 当前选中值 |
| `opened` | `bool` | 弹出层是否打开 |

## 方法详解

### `handle_event`（Widget）
- 检查输入框的 `TextInputAction::Return` 或 `TextInputAction::Up/Down` 时打开下拉
- 检查输入框的 `TextInputAction::Key`（键盘输入）时按首字母搜索匹配项并选中
- 检查选项按钮的点击事件，将其 `ButtonAction::Pressed` 转换为 `DropDownAction::Selected`
- 点击下拉打开/关闭按钮时切换弹出状态
- 点击弹出层外部时关闭下拉

### `draw_walk`（Widget）
- 先绘制背景和下拉箭头，然后绘制文本输入框
- 如果 `opened`，添加弹出层的绘制

### `set_options`
- 设置选项列表 `values` 并更新 PortalList 的数据范围

### `set_selected`
- 按值索引查找选项，更新 `input` 的显示文本和 `selected` 值

### `selected` / `changed`（DropDownRef）
- `selected`：检查 actions 中是否有 `DropDownAction::Selected`
- `changed`：检查 `DropDownAction::Selected` 且值不等于当前内部缓存值
