# `check_box.rs` — CheckBox / Toggle 复选框与开关控件

**文件路径**: `widgets/src/check_box.rs` (606 行)
**核心作用**: 实现 CheckBox（方块勾选框）、Toggle（滑动开关）及 ToggleFlat 三种 UI 控件。包含完整的渲染着色器、动画状态机、事件处理和脚本集成。

---

## 一、脚本 DSL 定义 (`script_mod!`)

### `DrawLabelText` — 自定义着色器组件 (L17-L21)

```rust
mod.widgets.DrawLabelTextBase = #(DrawLabelText::script_component(vm))
set_type_default() do #(DrawLabelText::script_shader(vm)) {
    ..mod.draw.DrawText  // 继承自 DrawText
}
```

### `CheckBoxBase` — 控件注册 (L16)

```rust
mod.widgets.CheckBoxBase = #(CheckBox::register_widget(vm))
```

### `CheckBoxFlat` — 扁平复选框变体 (L18-L227)

属性结构:
- **布局**: `width: Fit`, `height: Fit`, `padding: theme.mspace_2`, `align: Align{x: 0., y: 0.}`
- **标签**: `label_walk` 设置右边距以容纳复选框 (margin left: 13px)
- **Render**: `draw_bg` 使用 SDF 绘制, `draw_text` 用于标签
- **尺寸**: `size: uniform(15.0)`, `border_size/radius` 从 theme 取
- **着色器**: `pixel: fn()` 使用 `Sdf2d` 绘制方形背景和勾选标记

### `CheckBox` — 默认外观 (L229-L238)

继承 `CheckBoxFlat`，只覆盖 `border_color` 为 `theme.color_bevel_inset_1`（凹陷边框效果）

### `ToggleFlat` / `Toggle` — 滑动开关变体 (L240-L341)

`ToggleFlat` 继承 `CheckBoxFlat`，但替换 `draw_bg` 的 `pixel` 着色器为圆形/椭圆滑动开关渲染：
- 绘制胶囊形轨道 (pill)
- 绘制圆形滑块，固定位置 `mark_padding = 1.5`
- 滑块位置根据 `self.active` 插值 (`mark_target_y` 到 `mark_pos_y`)
- `Toggle` 在此基础上叠加凹陷边框效果

### Animator 配置 (L150-L226)

四个独立的动画轨道:

| 轨道 | 触发 | off→on | on→off | 效果 |
|------|------|--------|--------|------|
| `disabled` | 禁用 | Snap | 0.2s 淡入 | 透明灰色 |
| `hover` | 悬浮 | 0.15s | Snap | 高亮背景/文本 |
| `focus` | 键盘焦点 | Snap | Snap | 聚焦指示器 |
| `active` | 选中/未选中 | 0.1s | 0.0s | 勾选标记出现 |

`hover` 轨道还有 `down` 子状态 (0.2s 过渡)，按下时同时激活 hover + down。

### `CheckBoxFlat` → `ToggleFlat` 的像素着色器差异

CheckBox 使用 `sdf.box()` + `sdf.stroke()` 绘制方形勾选框和勾号路径，而 Toggle 使用:
- `sdf.box()` 绘制椭圆胶囊背景
- `sdf.circle()` + `sdf.subtract()` 绘制圆环（关闭时）/ 实心圆（开启时）
- 通过 `self.blend(self.active)` 控制 ring/fill 混合

---

## 二、Rust 结构体

### `CheckBox` (L363-L411)

```rust
#[derive(Script, Widget, Animator)]
pub struct CheckBox {
    #[uid]               uid: WidgetUid,
    #[source]            source: ScriptObjectRef,
    #[walk]              walk: Walk,
    #[layout]            layout: Layout,
    #[apply_default]     animator: Animator,
    #[live]              icon_walk: Walk,         // 图标布局约束
    #[live]              label_walk: Walk,        // 标签布局约束
    #[live]              label_align: Align,       // 标签对齐方式
    #[redraw] #[live]    draw_bg: DrawQuad,       // 复选框背景（可触发重绘）
    #[live]              draw_text: DrawText,      // 标签文本绘制
    #[live]              draw_icon: DrawSvg,      // 可选图标（SVG）
    #[live]              text: ArcStringMut,       // 标签文本内容
    #[visible] #[live(true)]  pub visible: bool,  // 可见性控制
    #[live(None)]        pub active: Option<bool>, // 选中状态（None=未初始化）
    #[live]              on_click: ScriptFnRef,    // 点击回调函数引用
    #[live]              bind: String,             // 数据绑定标识
    #[rust]              action_data: WidgetActionData, // action 数据存储
}
```

**关键字段详解**:

| 字段 | 类型 | 作用 | 脚本可设 | 默认值 |
|------|------|------|----------|--------|
| `uid` | `WidgetUid` | 全局唯一标识，事件路由/action 查找用 | 否 | 自动 |
| `source` | `ScriptObjectRef` | 脚本对象的引用，用于触发脚本回调 | 否 | 自动 |
| `walk` | `Walk` | 父容器布局约束 | 是 | 默认 |
| `layout` | `Layout` | 子元素布局参数 (flow, align, padding) | 是 | 默认 |
| `animator` | `Animator` | 动画状态机，控制所有过渡动画 | 是 | `@apply_default` |
| `draw_bg` | `DrawQuad` | 复选框/开关的 SDF 渲染四边形 | 是 | `#[redraw]` |
| `draw_text` | `DrawText` | 文本渲染器 | 是 | 默认 |
| `draw_icon` | `DrawSvg` | SVG 图标渲染器 | 是 | 默认 |
| `text` | `ArcStringMut` | 文本内容（写时复制，支持共享） | 是 | 空 |
| `visible` | `bool` | 控制可见性，false 则跳过绘制 | 是 | `true` |
| `active` | `Option<bool>` | None=未初始化，Some(true)=选中，Some(false)=未选中 | 是 | `None` |
| `on_click` | `ScriptFnRef` | 点击时调用的 Splash 脚本函数 | 是 | 空 |
| `bind` | `String` | 数据绑定的键名 | 是 | 空 |

### `CheckBoxAction` (L423-L428)

```rust
#[derive(Clone, Debug, Default)]
pub enum CheckBoxAction {
    Change(bool),   // 选中状态变更，bool=新状态
    #[default]
    None,
}
```

事件 action 类型。当用户点击切换时通过 `cx.widget_action_with_data()` 触发 `Change(new_state)`。

---

## 三、Trait 实现

### `ScriptHook for CheckBox` (L413-L421)

```rust
fn on_after_new(&mut self, vm: &mut ScriptVm) {
    // 如果 active 字段在脚本中有初始值，在初始化时同步到 animator
    if let Some(active) = self.active.take() {
        vm.with_cx_mut(|cx| {
            self.animator_toggle(cx, active, Animate::No, ids!(active.on), ids!(active.off));
        });
    }
}
```

**作用**: 当脚本引擎创建 CheckBox 新实例时调用。如果 `active` 在 DSL 中有初始值（如 `active: true`），将该值同步到 animator 的 `active` 动画状态。

**注意**: 使用 `Animate::No` 而非 `Animate::Yes`，因为脚本重载时必须强制应用新状态（防止 animator 缓存的状态导致新旧值不一致）。

### `Widget for CheckBox` (L473-L570)

#### `set_disabled` / `disabled` — 禁用状态控制

```rust
fn set_disabled(&mut self, cx: &mut Cx, disabled: bool)
fn disabled(&self, cx: &Cx) -> bool
```

切换 `disabled` 动画轨道，控制 `draw_bg` 和 `draw_text` 的 `disabled` uniform 值（0.0→1.0）。

#### `script_call` — 脚本方法调用

```rust
fn script_call(&mut self, vm: &mut ScriptVm, method: LiveId, _args: ScriptValue) -> ScriptAsyncResult
```

支持 Splash 脚本调用 `checked()` 方法，返回当前是否选中 (`animator_in_state(cx, ids!(active.on))`)。

#### `handle_event` — 事件处理 (L501-L553)

完整的事件处理流程：

```
收到事件
  ↓
self.animator_handle_event(cx, event)  →  驱动动画
  ↓
event.hits(cx, self.draw_bg.area())  →  判断命中的事件类型
  ├── KeyFocus(_)           → 播放 focus.on 动画
  ├── KeyFocusLost(_)       → 播放 focus.off 动画 + 重绘
  ├── FingerHoverIn(_)      → cursor = Hand + 播放 hover.on
  ├── FingerHoverOut(_)     → 播放 hover.off
  ├── FingerDown(fe)        → 
  │    1. self.set_key_focus(cx)       — 获取键盘焦点
  │    2. 检查当前 active 状态：
  │       ├── active.on   → 播放 active.off + 触发 Change(false) action
  │       └── active.off  → 播放 active.on  + 触发 Change(true) action
  │    3. cx.widget_to_script_call()   — 调用 on_click 回调
  └── FingerUp/FingerMove  → 空实现（无拖动逻辑）
```

**关键设计点**:
- 每次点击都反转状态（toggle 行为），而非长按
- 点击立即触发 `on_click` 脚本回调，传入新状态值作为参数
- 使用 `widget_action_with_data` 携带 `CheckBoxAction` 和 `action_data`

#### `draw_walk` — 绘制方法 (L555-L560)

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, _scope: &mut Scope, walk: Walk) -> DrawStep
```

- 如果 `!self.visible`，直接返回 `DrawStep::done()`（跳过绘制）
- 否则委托给 `self.draw_check_box(cx, walk)`

#### `text` / `set_text` — 文本操作

```rust
fn text(&self) -> String
fn set_text(&mut self, cx: &mut Cx, v: &str)
```

`set_text` 使用 `as_mut_empty()` 避免不必要的分配，然后 `push_str` 复制文本，最后请求重绘。

---

## 四、方法详解

### `CheckBox` impl 块 (L430-L471)

#### `draw_check_box` — 核心绘制逻辑 (L431-L441)

```rust
pub fn draw_check_box(&mut self, cx: &mut Cx2d, walk: Walk) -> DrawStep {
    self.draw_bg.begin(cx, walk, self.layout);   // 开始 Turtle 布局
    self.draw_icon.draw_walk(cx, self.icon_walk); // 绘制 SVG 图标
    self.draw_text.draw_walk(                     // 绘制标签文本
        cx, self.label_walk, self.label_align, self.text.as_ref()
    );
    self.draw_bg.end(cx);                         // 结束 Turtle 布局
    cx.add_nav_stop(self.draw_bg.area(), NavRole::TextInput, Inset::default());
    DrawStep::done()
}
```

绘制顺序: 背景(含 SDF 复选框) → SVG 图标 → 文本标签。
最后注册键盘导航停止点 (`NavRole::TextInput`)。

#### `changed` — 检查 action (L443-L450)

```rust
pub fn changed(&self, actions: &Actions) -> Option<bool>
```

在 `actions` 列表中查找当前 widget 的 `CheckBoxAction::Change`，返回 `Some(new_state)` 或 `None`。

#### `active` — 当前选中状态 (L452-L454)

```rust
pub fn active(&self, cx: &Cx) -> bool
```

委托给 `self.animator_in_state(cx, ids!(active.on))`，检查 animator `active` 轨道是否在 `on` 状态。

#### `set_active` — 设置选中状态 (L464-L466)

```rust
pub fn set_active(&mut self, cx: &mut Cx, value: bool, animate: Animate)
```

- `Animate::Yes`: 使用 `animator_toggle` 播放过渡动画（0.1s）
- `Animate::No`: 用于脚本重载时强制 cut 到新状态（防止缓存导致的 stale uniform）

#### `debug_dump_animator` — 调试 (L468-L470)

```rust
pub fn debug_dump_animator(&self, heap: &ScriptHeap) -> String
```

### `CheckBoxRef` impl 块 (L572-L605)

`CheckBoxRef` 是通过 `WidgetRef` 宏生成的智能指针，提供线程/借用安全的跨组件访问。

| 方法 | 底层调用 | 说明 |
|------|----------|------|
| `changed(actions)` | `inner.borrow().changed(actions)` | 只读检查 action |
| `set_text(text)` | `inner.borrow_mut().set_text()` | 内部使用 `as_mut_empty` 复用缓冲区 |
| `active(cx)` | `inner.borrow().active(cx)` | 只读查询状态 |
| `set_active(cx, val, animate)` | `inner.borrow_mut().set_active()` | 设置状态 |
| `debug_dump_animator(heap)` | `inner.borrow().debug_dump_animator()` | 调试输出 |

所有 `CheckBoxRef` 方法都处理 `borrow()/borrow_mut()` 失败的情况（返回默认值或静默忽略），不会 panic。

---

## 五、着色器实现（SDF 绘制）

### CheckBox 勾选框像素着色器 (L64-L118)

```
流程:
1. 计算绘制区域: sz_px=15, center_px, offset_px
2. sdf.box() 绘制圆角方形背景
3. sdf.fill_keep(color_fill) — 填充保留（后续可描边）
4. sdf.stroke(color_stroke, border_size) — 描边框
5. sdf.move_to / line_to — 绘制勾选符号路径（三条线组成 ✓）
6. sdf.stroke(mark_color, size*0.09) — 渲染勾选符号
```

颜色混合链:
```
color_fill = self.color
    .mix(self.color_focus, self.focus)
    .mix(self.color_active, self.active)
    .mix(self.color_hover, self.hover)
    .mix(self.color_down, self.down)
    .mix(self.color_disabled, self.disabled)
```

### Toggle 滑动开关像素着色器 (L252-L306)

```
流程:
1. 计算绘制区域: sz_px = (size*1.6, size) — 宽高比 1.6:1
2. sdf.box() 绘制胶囊形轨道 (圆角 = bonus*size*0.1)
3. sdf.fill_keep + stroke 填充并描边
4. 滑块位置计算:
   mark_target_y = sz_px.y - sz_px.x + border_size + mark_padding
   mark_pos_y = sz_px.y*0.5 + border_size - mark_target_y * self.active
   → self.active=0 时滑块在左侧，=1 时在右侧
5. sdf.circle() + subtract 绘制滑块的圆环/实心圆效果
6. sdf.blend(self.active) 控制 ring/fill 混合比例
```

---

## 六、与类似控件的比较

| 特性 | `CheckBox` | `Toggle` / `ToggleFlat` |
|------|------------|------------------------|
| 外观 | 方形勾选框 | 胶囊形滑动开关 |
| 选中指示 | ✓ 勾号图形 | 滑块从左→右滑动 |
| DSL 名称 | `CheckBoxFlat` / `CheckBox` | `ToggleFlat` / `Toggle` |
| 基类 | `CheckBoxBase` (Rust struct) | 直接派生于 `CheckBoxFlat` |
| 行为 | 完全一致（相同 handle_event） | 完全一致 |
| 动画 | `active` 轨道 0.1s | `active` 轨道 0.1s + `ease: OutQuad` |
