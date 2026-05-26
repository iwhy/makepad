# `drop_down.rs` — DropDown 下拉选择控件

**文件路径**: `widgets/src/drop_down.rs` (767 行)
**核心作用**: 实现 DropDown 下拉选择框，包含按钮区域渲染、PopupMenu 弹出菜单集成、键盘导航（↑↓方向键选择）、以及多种外观变体（平面/渐变/横纵渐变）。

---

## 一、脚本 DSL 定义 (`script_mod!`)

### `DrawLabelText` — 自定义文本着色器组件 (L17-L21, L411-L420)

```rust
#[repr(C)]
struct DrawLabelText {
    #[deref] draw_super: DrawText,  // 继承 DrawText 所有字段
    #[live]  focus: f32,            // 焦点状态 uniform
    #[live]  hover: f32,            // 悬浮状态 uniform
}
```

通过 `#[deref]` 继承 DrawText 的所有字段，额外添加 `focus`/`hover` 两个 uniform。注册为 `script_shader` 和 `script_component` 两种角色。

### `DropDownBase` / `DropDownFlat` — 层次体系 (L18-L302)

```
DropDownBase → DropDownFlat → DropDown → DropDownGradientY → DropDownGradientX
                                    ↘ DropDownFlat (直接变体)
```

| 变体 | 特点 |
|------|------|
| `DropDownFlat` | 平面外观，带箭头三角形 + 渐变填充/描边支持 |
| `DropDown` | 凸出边框 (`bevel_outset_1`)，使用 `PopupMenuFlat` |
| `DropDownGradientY` | 垂直渐变色 (`color_2`)，使用 `PopupMenuGradientY` |
| `DropDownGradientX` | 水平渐变 (`gradient_border_horizontal: 1.0`)，使用 `PopupMenuGradientX` |

### DropDownFlat 着色器绘制 (L113-L229)

箭头三角形绘制 (L130-L158):
```
sdf.move_to(c.x-sz-offset_x, c.y-sz+offset)
sdf.line_to(c.x+sz-offset_x, c.y-sz+offset)
sdf.line_to(c.x-offset_x, c.y+sz*0.25+offset)
sdf.close_path()
sdf.fill_keep(arrow_color_mixed)
```

渐变色填充/描边支持 (L197-L226):
- `self.color_2.x > -0.5` 时启用渐变填充（混合 color 和 color_2）
- `self.border_color_2.x > -0.5` 时启用渐变描边
- 渐变方向由 `gradient_border_horizontal` / `gradient_fill_horizontal` 控制
- 使用 `dither = Math.random_2d(...) * 0.04` 实现抗锯齿抖动

### PopupMenu 嵌入 (L232-L233)

```rust
popup_menu: mod.widgets.PopupMenu{}
```

每个 DropDown 变体关联不同的 PopupMenu 变体，从 DSL 配置。

---

## 二、Rust 枚举与结构体

### `PopupMenuPosition` (L352-L358)

```rust
#[repr(C)]
pub enum PopupMenuPosition {
    #[pick]
    OnSelected,     // 弹出菜单位置跟随当前选中项（默认）
    BelowInput,     // 弹出菜单位置在输入框下方
}
```

控制弹出菜单的屏幕位置。

### `DrawLabelText` (L411-L420)

```rust
#[repr(C)]
struct DrawLabelText {
    #[deref] draw_super: DrawText,
    #[live]  focus: f32,
    #[live]  hover: f32,
}
```

自定义 DrawText 子类，添加 `focus`/`hover` 动画 uniform。

### `PopupMenuGlobal` (L406-L409)

```rust
struct PopupMenuGlobal {
    map: Rc<RefCell<ComponentMap<ScriptValue, PopupMenu>>>,
}
```

全局共享的 PopupMenu 缓存映射。通过 `cx.global::<PopupMenuGlobal>()` 访问，实现在 `DropDown` 和 `PopupMenu` 之间的共享状态。

### `DropDownAction` (L449-L454)

```rust
pub enum DropDownAction {
    Select(usize),    // 选中索引为 usize 的项
    #[default] None,
}
```

### `DropDown` (L360-L404)

```rust
pub struct DropDown {
    #[uid]       uid: WidgetUid,
    #[source]    source: ScriptObjectRef,
    #[apply_default] animator: Animator,
    #[redraw] #[live] draw_bg: DrawQuad,        // 背景+箭头四边形
    #[live]      draw_text: DrawLabelText,       // 文本渲染(带 focus/hover)
    #[walk]      walk: Walk,
    #[live]      bind: String,                   // 数据绑定键
    #[live]      bind_enum: String,              // 枚举绑定键
    #[live]      popup_menu: ScriptValue,         // PopupMenu 引用(脚本值)
    #[live]      labels: Vec<String>,             // 下拉选项标签列表
    #[live]      popup_menu_position: PopupMenuPosition,  // 菜单位置
    #[rust]      is_active: bool,                 // 下拉是否展开
    #[live]      selected_item: usize,            // 当前选中索引
    #[layout]    layout: Layout,
    #[rust]      action_data: WidgetActionData,
}
```

**关键字段**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `popup_menu` | `ScriptValue` | 脚本中的 PopupMenu 对象引用，延迟实例化 |
| `labels` | `Vec<String>` | 选项文本列表，通过 DSL 或 `set_labels` 设置 |
| `selected_item` | `usize` | 当前选中索引，配合 labels 渲染文本 |
| `is_active` | `bool` | 菜单是否展开，控制弹出菜单的渲染和事件处理 |
| `popup_menu_position` | `PopupMenuPosition` | 弹出菜单相对位置 |

---

## 三、Trait 实现

### `ScriptHook for DropDown` (L422-L447)

```rust
fn on_after_apply(&mut self, vm, apply, scope, obj) {
    // 从 popup_menu 脚本值创建/获取 PopupMenu 实例
    // 使用 cx.global::<PopupMenuGlobal>() 缓存映射
    // 使用 try_borrow_mut 避免嵌套 on_after_apply 导致的 panic
}
```

在脚本应用属性后调用，确保 `popup_menu` 引用被正确解析为 `PopupMenu` 实例，并缓存到全局映射。

### `Widget for DropDown` (L549-L673)

#### `handle_event` — 事件处理 (L564-L668)

复杂的事件处理流程，分为两个阶段:

**阶段 1: 弹出菜单事件处理** (L568-L603)
- 如果 `is_active` 且 `popup_menu` 非空:
  - 通过 `menu.handle_event_with()` 处理 PopupMenu 内部事件
  - `PopupMenuAction::WasSelected` → 更新 `selected_item` + 触发 `DropDownAction::Select` + 关闭菜单
  - 全局鼠标点击事件检查: 点击弹出菜单外部区域时关闭菜单

**阶段 2: 按钮自身事件处理** (L606-L667)
- `KeyFocusLost` → 关闭菜单 + 关闭 focus/hover 动画
- `KeyFocus` → 播放 focus.on 动画
- `KeyDown(ArrowUp)` → 选中项减 1 + 触发 action
- `KeyDown(ArrowDown)` → 选中项加 1 + 触发 action
- `FingerDown` → 设置键盘焦点 + 播放按下动画 + `set_active()` 展开菜单
- `FingerHoverIn` → cursor = Hand + 播放 hover.on
- `FingerHoverOut` → 播放 hover.off

#### `draw_walk` — 绘制方法 (L670-L673)

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, _scope: &mut Scope, walk: Walk) -> DrawStep
```

委托给 `self.draw_walk(cx, walk)` 并返回 `DrawStep::done()`。

注意: `draw_walk` 方法名与内部方法 `draw_walk` (L492) 同名，Rust 解析器区分 trait 方法和 impl 方法。

### 内部 `draw_walk` — 下拉按钮绘制 (L492-L546)

```rust
pub fn draw_walk(&mut self, cx: &mut Cx2d, walk: Walk)
```

1. 绘制背景四边形 + 文本标签（当前选中项或空格占位）
2. 注册键盘导航停止点 (`NavRole::DropDown`)
3. 如果弹出菜单展开，按位置策略绘制 PopupMenu 项:
   - `OnSelected`: 偏移到当前选中项位置
   - `BelowInput`: 偏移到按钮底部

---

## 四、方法详解

### `DropDown` impl (L456-L547)

| 方法名 | 签名 | 说明 |
|--------|------|------|
| `selected_item_index` | `() -> usize` | 返回当前选中索引 |
| `selected_item_label` | `() -> String` | 返回当前选中标签文本 |
| `set_active` | `(&mut self, cx: &mut Cx)` | 展开弹出菜单，初始化选中项，sweep_lock |
| `set_closed` | `(&mut self, cx: &mut Cx)` | 关闭菜单，sweep_unlock |
| `draw_text` | `(&mut self, cx: &mut Cx2d, label: &str)` | 仅绘制文本部分（用于某些定制场景） |
| `draw_walk` | `(&mut self, cx: &mut Cx2d, walk: Walk)` | 完整绘制下拉按钮 + 弹出菜单 |

**`set_active` 实现细节**:
1. 设置 `is_active = true`
2. 请求重绘
3. 从全局映射获取 PopupMenu 实例
4. 调用 `lb.init_select_item(node_id)` 初始化选中项
5. 调用 `cx.sweep_lock(self.draw_bg.area())` 锁定事件区域（防止外部点击穿透）

**`set_closed` 实现细节**:
1. 设置 `is_active = false`
2. 请求重绘
3. 调用 `cx.sweep_unlock(self.draw_bg.area())` 解锁事件区域

### `DropDownRef` impl (L676-L767)

`DropDownRef` 是线程/借用安全的跨组件访问包装。

| 方法 | 说明 |
|------|------|
| `set_labels_with(cx, f)` | 通过闭包设置标签列表，自动增减 Vec 大小 |
| `set_labels(cx, labels)` | 直接设置标签列表 |
| `selected(actions)` | 检查当前是否选中某项（返回 `Option<usize>`） |
| `changed(actions)` | 同上，别名 |
| `changed_label(actions)` | 返回选中项的标签文本 |
| `set_selected_item(cx, item)` | 程序化设置选中项（含 bounds check） |
| `selected_item()` | 返回当前选中索引 |
| `selected_label()` | 返回当前选中标签文本 |
| `set_selected_by_label(label, cx)` | 通过标签文本设置选中项 |

---

## 五、Animator 动画配置 (L236-L301)

| 轨道 | 过渡 | 作用 |
|------|------|------|
| `disabled` | off: Snap / on: 0.2s | 禁用/启用过渡 |
| `hover` | off: 0.1s / on: 0.1s(down: 0.01s) / down: 0.2s | 悬浮/按下状态 |
| `focus` | off: 0.2s / on: 0.0s | 键盘焦点过渡 |

hover 的 `down` 子状态使用 `[{time: 0.0, value: 1.0}]` 语法实现瞬间跳转。

---

## 六、与其他控件的协作

| 组件 | 关系 |
|------|------|
| `PopupMenu` | DropDown 的核心弹出菜单，通过 `handle_event_with` 委托事件 |
| `PopupMenuGlobal` | 全局缓存 PopupMenu 实例，避免重复创建 |
| `crate::animator::*` | 动画状态机驱动 |
