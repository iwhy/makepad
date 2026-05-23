# expandable_panel.rs — ExpandablePanel 可折叠/展开面板

## 文件概述

`ExpandablePanel` 是 Makepad 的可折叠/展开面板组件。用户点击面板头部可切换内容区域的展开和折叠状态，常用于设置面板、侧边栏分组、FAQ 列表等场景。

---

## `ExpandablePanel` 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct ExpandablePanel {
    #[source] source: ScriptObjectRef,
    #[deref] view: View,
    #[live] animator: Animator,
    #[live] header: View,                    // 面板头部（始终可见）
    #[live] content: View,                   // 面板内容（可折叠）
    #[live] expanded: bool,                  // 当前是否展开
    #[live] expand_icon: Option<WidgetRef>,  // 展开/折叠图标
    #[live] header_height: f64,              // 头部高度
    #[live] animated: bool,                  // 是否使用动画过渡
}
```

---

## 核心逻辑

### 展开/折叠

```
点击头部 → 切换 expanded 状态
  ├─ true → 展开：content 可见，高度从 0 动画到 content_size
  └─ false → 折叠：content 隐藏，高度从 content_size 动画到 0
```

```rust
fn handle_header_click(&mut self, cx) {
    self.expanded = !self.expanded;
    
    if self.animated {
        // 启动 Animator 中的展开/折叠动画
        let state = if self.expanded { "expand" } else { "collapse" };
        self.animator.play(cx, live_id!(state));
    }
}
```

### 展开状态绘制

`draw_walk` 中根据 `expanded` 状态决定是否绘制 `content`：

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    while self.view.draw_walk(cx, scope, walk).step() {
        // 1. 始终绘制 header
        self.header.draw_walk(cx, scope, walk);
        
        // 2. 根据 expanded 状态绘制 content
        if self.expanded || self.animator.is_active() {
            // 动画过渡期间，限制 content 高度
            let content_height = if self.animated {
                // 动画插值高度
                self.content_size * self.anim_progress
            } else {
                // 完全展开
                self.content_size
            };
            
            cx.turtle().push_height(content_height);
            self.content.draw_walk(cx, scope, walk);
            cx.turtle().pop_height();
        }
    }
}
```

### 高度动画

当 `animated: true` 时，展开/折叠使用 `anim_timeline` 实现平滑的高度过渡：

- 展开：`anim_progress` 从 0 → 1，`content_height` 从 0 → `full_height`。
- 折叠：`anim_progress` 从 1 → 0，`content_height` 从 `full_height` → 0。

动画持续期间两个都会绘制，但 content 被裁剪到当前的动画高度。

---

## 事件处理

```rust
fn handle_event(&mut self, cx, event, scope) {
    // 1. header 的事件处理（点击展开/折叠）
    self.header.handle_event(cx, event, scope);
    
    // 2. content 仅在展开时处理事件
    if self.expanded {
        self.content.handle_event(cx, event, scope);
    }
}
```

当面板折叠时，content 区域不处理任何鼠标事件（事件不会穿透到隐藏的 content）。

---

## 使用场景

| 场景 | 模式 |
|------|------|
| 设置面板 | animated: true，分组可折叠选项 |
| 侧边栏分组 | animated: true，文件/功能分组 |
| FAQ 列表 | animated: true，点击问题展开答案 |
| 调试面板 | animated: false，即时展开/折叠 |

---

## 与 FoldButton/FoldHeader 的关系

- **FoldButton**：独立的折叠按钮（三角箭头图标），不附带面板。
- **FoldHeader**：折叠头部组件，包含图标和文本。
- **ExpandablePanel**：集成了 FoldButton + FoldHeader + 可折叠内容面板的完整组件。

```
ExpandablePanel
  ├── header (FoldHeader)
  │     ├── FoldButton (▶/▼ 图标)
  │     └── Label (标题文本)
  └── content (View)
        └── 任意子组件
```
