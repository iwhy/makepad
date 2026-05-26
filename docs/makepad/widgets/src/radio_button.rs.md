# `radio_button.rs` — RadioButton 单选按钮控件

**文件路径**: `widgets/src/radio_button.rs` (516 行)
**核心作用**: 实现 RadioButton（圆形单选按钮）及其变体 RadioButtonTab（选项卡式）和 RadioButtonFlatter（超精简）。包含 SDF 圆形渲染、互斥选择逻辑、动画状态机和完整的事件处理。

---

## 一、脚本 DSL 定义 (`script_mod!`)

### 控件继承体系 (L14-L292)

```
RadioButtonBase → RadioButtonFlat → RadioButton → RadioButtonFlatter
                                    ↘ RadioButtonTabFlat → RadioButtonTab
```

| 变体 | 外观 | 继承 |
|------|------|------|
| `RadioButtonFlat` | 圆形单选按钮 + 文本标签（平面） | `RadioButtonBase` (Rust struct) |
| `RadioButton` | 叠加凹陷 (`bevel_inset_1`) 边框 | `RadioButtonFlat` |
| `RadioButtonFlatter` | 隐藏圆形，仅文本 + 颜色变化 | `RadioButton` |
| `RadioButtonTabFlat` | 方形选项卡外观 | `RadioButtonBase` |
| `RadioButtonTab` | 叠加凸出 (`bevel_outset_1`) 边框 | `RadioButtonTabFlat` |

### RadioButtonFlat 着色器 (L31-L97)

SDF 圆形绘制:
```
1. sdf.circle(center_x, center_y, radius - border_size)
2. sdf.fill_keep(color_fill) + sdf.stroke(color_stroke, border_size)
3. sdf.circle(center_x, center_y + mark_offset, radius*0.5 - border_size*0.75)
4. sdf.fill(mark_color)
```

内圆点偏移 `mark_offset` 可用于实现脉冲/选中效果动画。

### RadioButtonTabFlat 着色器 (L248-L273)

方形选项卡外观，使用 `sdf.box()` 代替圆形:
```
1. sdf.box(border_size, border_size, w-2*border, h-2*border, border_radius)
2. fill_keep + stroke
```

### Animator 动画配置 (L127-L203)

| 轨道 | off→on | on→off | 作用 |
|------|--------|--------|------|
| `disabled` | 0.0s Snap | 0.2s | 禁用 |
| `hover` | 0.15s | Snap (down: 0.2s) | 悬浮/按下 |
| `active` | 0.2s | 0.0s | 选中 |
| `focus` | 0.2s | 0.0s | 键盘焦点 |

---

## 二、Rust 枚举与结构体

### `RadioButtonAction` (L295-L300)

```rust
pub enum RadioButtonAction {
    Clicked,        // 单选按钮被选中
    #[default] None,
}
```

### `RadioButton` (L302-L343)

```rust
pub struct RadioButton {
    #[uid]       uid: WidgetUid,
    #[source]    source: ScriptObjectRef,
    #[walk]      walk: Walk,
    #[layout]    layout: Layout,
    #[apply_default] animator: Animator,
    #[live]      icon_walk: Walk,       // SVG 图标布局
    #[live]      label_walk: Walk,      // 标签布局
    #[live]      label_align: Align,    // 标签对齐
    #[redraw] #[live] draw_bg: DrawQuad, // 单选圆圈背景
    #[live]      draw_text: DrawText,    // 标签文本
    #[live]      draw_icon: DrawSvg,     // 图标
    #[live]      text: ArcStringMut,     // 标签内容
    #[visible] #[live(true)] pub visible: bool,
    #[live]      bind: String,
    #[rust]      action_data: WidgetActionData,
}
```

与 `CheckBox` 结构几乎一致，但无 `active: Option<bool>` 和 `on_click` 回调——`RadioButton` 的选中由 `RadioButtonSet` 管理，不自动初始化。

---

## 三、Trait 实现

### `Widget for RadioButton` (L381-L452)

#### `handle_event` — 事件处理 (L396-L435)

```
event.hits(cx, self.draw_bg.area())
  ├── KeyFocus(_)           → play focus.on
  ├── KeyFocusLost(_)       → play focus.off + redraw
  ├── FingerHoverIn(_)      → cursor = Hand + play hover.on
  ├── FingerHoverOut(_)     → cursor = Arrow + play hover.off
  ├── FingerDown(fe)        → play hover.down + set_key_focus
  ├── FingerUp(_fe)         → play hover.on + 如果 active.off 则：
  │                             1. play active.on
  │                             2. cx.widget_action(Click)
  └── FingerMove → noop
```

**关键差异**（对比 CheckBox）:
- 单选按钮只能在 `active.off` → `active.on`（不能反向切换）
- 切换逻辑在 `FingerUp` 而非 `FingerDown`（避免误触发）
- 没有 `on_click` 脚本回调

#### `draw_walk` (L437-L442)

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    if !self.visible { return DrawStep::done() }
    self.draw_radio_button(cx, walk)
}
```

---

## 四、方法详解

### `RadioButton` impl (L345-L379)

| 方法 | 签名 | 说明 |
|------|------|------|
| `draw_radio_button` | `(&mut self, cx, walk) -> DrawStep` | 核心绘制：draw_bg → icon → text → nav_stop |
| `clicked` | `(&self, actions) -> bool` | 检查 actions 中是否有 Clicked 事件 |
| `active` | `(&self, cx) -> bool` | `animator_in_state(cx, ids!(active.on))` |
| `set_active` | `(&mut self, cx, value, animate)` | 设置选中态，`Animate::No` 用于脚本重载 |

### `RadioButtonRef` impl (L454-L498)

| 方法 | 说明 |
|------|------|
| `clicked(actions)` | 检查本按钮是否被点击 |
| `unselect(cx)` | 切换到未选中 (`animator_play(active.off)`) |
| `select(cx, scope)` | 切换到选中并触发 action |
| `set_text(text)` | 设置标签文本 |
| `active(cx)` | 查询选中状态 |
| `set_active(cx, val, animate)` | 设置选中状态 |

### `RadioButtonSet` impl (L501-L515)

```rust
pub fn selected(&self, cx: &mut Cx, actions: &Actions) -> Option<usize>
```

**核心互斥逻辑**:
1. 遍历所有 RadioButton 实例
2. 找到被点击的那个 (`item.clicked(actions)`)
3. 取消选中其他所有按钮 (`other.unselect(cx)`)
4. 返回被选中按钮的索引

这个方法是 `RadioButtonSet` 独有的——`RadioButtonSet` 是由 `WidgetSet` derive 生成的集合类型，包含同一父容器下所有 RadioButton 的引用。

---

## 五、变体对比

| 变体 | 渲染方式 | 文本样式 | 适用场景 |
|------|----------|----------|----------|
| `RadioButtonFlat` | 圆形 SDF | `color_label_outer` | 表单中的选项列表 |
| `RadioButton` | 圆形 + 凹陷边框 | `color_label_outer` | 带立体感的表单 |
| `RadioButtonFlatter` | 无圆形（透明） | `color_label_outer_off/active` | 仅文本差异的极简设计 |
| `RadioButtonTabFlat` | 方形 SDF | `color_label_inner/active` | 选项卡式导航 |
| `RadioButtonTab` | 方形 + 凸出边框 | `color_label_inner/active` | 带立体感的选项卡 |
