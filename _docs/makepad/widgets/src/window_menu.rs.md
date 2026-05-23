# window_menu.rs — WindowMenu 窗口菜单栏组件

## 文件概述

`WindowMenu` 是 Makepad 窗口顶部的菜单栏组件，提供类似桌面应用的菜单系统，包含菜单项、子菜单和快捷键提示。

---

## `WindowMenu` 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct WindowMenu {
    #[deref] view: View,
    #[live] items: Vec<MenuItem>,           // 菜单项列表
    #[rust] open_menu_index: Option<usize>, // 当前展开的菜单索引
}
```

### `MenuItem`

```rust
pub struct MenuItem {
    pub id: LiveId,            // 菜单项标识符
    pub title: String,         // 显示文本
    pub shortcut: Option<String>, // 快捷键提示
    pub enabled: bool,          // 是否可点击
    pub sub_items: Vec<MenuItem>, // 子菜单项
}
```

---

## 菜单交互

### 点击展开

1. 鼠标点击菜单栏中的某个菜单标题（如 "File"、"Edit"）。
2. `open_menu_index` 设置为该菜单的索引。
3. 弹出下拉列表。

### 子菜单导航

1. 鼠标悬停在一个带有 `sub_items` 的菜单项上。
2. 展开次级子菜单（层级嵌套）。
3. 鼠标移出子菜单区域后自动关闭。

### 菜单选择

1. 点击菜单项触发对应的 Action。
2. Action 通过 `WidgetAction` 机制发送到应用。
3. 菜单关闭。

---

## 快捷键提示

每个 `MenuItem` 的 `shortcut` 字段显示快捷键提示文本：

```
File ──────────────────────────
  New                     Ctrl+N
  Open                    Ctrl+O
  ──────────────────────────
  Save                    Ctrl+S
  Save As...        Ctrl+Shift+S
  ──────────────────────────
  Exit                    Ctrl+Q
```

- 分隔线通过空标题或特殊 id 实现。
- 快捷键文本右对齐。
- 实际快捷键处理不由 WindowMenu 负责，而是 Window 级别的键盘监听。

---

## 菜单打开/关闭逻辑

```rust
fn handle_event(&mut self, cx, event, scope) {
    match event {
        Event::MouseDown(e) => {
            // 1. 检查点击是否在某个菜单标题或展开的菜单项上
            // 2. 如果在子菜单区域外点击，关闭所有菜单
            // 3. 如果在菜单项上，触发 action 并关闭
        }
        Event::MouseMove(e) => {
            // 1. 当有菜单打开时，悬停到另一个菜单标题自动切换
            // 2. 悬停到带子菜单的项时自动展开子菜单
        }
    }
}
```

关闭条件：
- 点击菜单外的区域。
- 点击菜单项触发 action。
- 按下 Esc 键。
