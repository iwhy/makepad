# app.rs

应用程序主文件 — UI Zoo 的入口和核心逻辑。

## 宏与模块注册

```rust
use makepad_widgets::*;
app_main!(App);
```

`app_main!` 宏生成 `main()` 函数和 Makepad 运行时入口。

## script_mod — UI 定义

### 布局模板

```rust
let UIZooTab = RectView{ height: Fill width: Fill flow: Down padding: 0 spacing: 0. }
```

`UIZooTab` 是每个 Demo Tab 的基础容器。使用 `RectView`（纯矩形，无背景装饰），垂直排列。

### Dock 系统（核心 UI）

```rust
let AppDock = Dock{
    root := DockSplitter{
        axis: SplitterAxis.Horizontal
        a: @tab_set_1
        b: @tab_set_2
    }
```

整个 UI Zoo 使用 **Dock 系统** 构建，这是 Makepad 中最复杂的布局容器：

- **`Dock`**: 顶层容器，管理整套分屏/标签系统。
- **`DockSplitter`**: 分隔器，将 Dock 拆分为左右两部分。
  - `axis: SplitterAxis.Horizontal` — 水平分割（左右布局）。
  - `align: SplitterAlign.FromA(0.0)` — 从 A 侧开始，比例为 0（即左侧初始为最小宽度）。
  - `a: @tab_set_1` / `b: @tab_set_2` — 左右两个分屏区域。

```rust
tab_set_1 := DockTabs{ tabs: [@tab_a] selected: 0 closable: false }
tab_set_2 := DockTabs{ tabs: [@tOverview, @tLayoutDemos, ...] selected: 0 closable: false }
```

左侧（`tab_set_1`）只包含欢迎标签页。右侧（`tab_set_2`）包含所有 Demo 标签页。

### DockTab 定义

```rust
tab_a := DockTab{ name: "Welcome" template: @PermanentTab kind: @TabOverview }
tButton := DockTab{ name: "Button" template: @PermanentTab kind: @TabButton }
// ... 共 26+ 个 DockTab
```

每个标签页使用 `DockTab`:
- `name`: 标签页标题
- `template: @PermanentTab` — 永久标签页（不可关闭）
- `kind`: 指向具体的 Tab 模板名称

### Tab 内容模板

```rust
TabButton := UIZooTab{DemoButton{}}
TabCheckBox := UIZooTab{DemoCheckBox{}}
// ... 每个 Demo 一个
```

`UIZooTab` 容器包裹具体的 Demo 组件。`DemoButton`、`DemoCheckbox` 等都在各自的模块文件中通过 `script_mod!` 注册到 `mod.widgets` 命名空间。

```rust
mod.gc.set_static(AppDock)
mod.gc.run()
```

将 `AppDock` 设为静态垃圾回收根，并运行 GC 初始化。

### startup 块

```rust
startup() do #(App::script_component(vm)){
    ui: Root{
        main_window := Window{
            window.inner_size: vec2(1200 800)
            body +: { flow: Down spacing: 0. margin: 0. dock := AppDock{} }
        }
    }
}
```

启动时创建窗口（1200×800），将 `AppDock` 实例化放入窗口 body。

## App 结构体

```rust
#[derive(Script, ScriptHook)]
pub struct App {
    #[live] ui: WidgetRef,
    #[rust] counter: usize,
}
```

- `ui`: 通过 `WidgetRef` 持有对根 UI 树的引用。
- `counter`: 用于 Demo 交互的计数器。

## handle_actions — 事件处理

所有跨多个 Demo 的交互事件都集中在 `MatchEvent::handle_actions` 中处理。

### RadioButton 组

```rust
ui.radio_button_set(cx, ids_array!(radios_demo_1.radio1, radios_demo_1.radio2, ...)).selected(cx, actions);
```

5 组 RadioButton 组用 `radio_button_set().selected()` 实现互斥选择。

### TextInput 响应

```rust
if let Some(txt) = self.ui.text_input(cx, ids!(simpletextinput)).changed(&actions) {
    self.counter += 1;
    let lbl = self.ui.label(cx, ids!(simpletextinput_outputbox));
    lbl.set_text(cx, &format!("{} {}", self.counter, txt));
}
```

监听文本输入变化，将输入内容实时更新到输出 Label。

### Multiline Toggle

```rust
if let Some(is_multiline) = self.ui.check_box(cx, ids!(multiline_toggle)).changed(actions) {
    let ti = self.ui.text_input(cx, ids!(multiline_toggleable));
    ti.set_is_multiline(cx, is_multiline);
}
```

勾选/取消勾选 CheckBox 切换 TextInput 的多行模式。

### Button 点击

```rust
if self.ui.button(cx, ids!(basicbutton)).clicked(&actions) {
    // 更新按钮文字
}
```

`basicbutton` 和 `iconbutton` 点击后更新文字内容。

### ImageBlend

```rust
if self.ui.button(cx, ids!(blendbutton)).clicked(&actions) {
    self.ui.image_blend(cx, ids!(blendimage)).switch_image(cx);
}
```

点击按钮切换 ImageBlend 的显示图像。

### PageFlip

```rust
if self.ui.button(cx, ids!(pageflipbutton_a)).clicked(&actions) {
    self.ui.page_flip(cx, ids!(page_flip)).set_active_page(cx, live_id!(page_a));
}
```

三个按钮各导航到 PageFlip 的不同页。

### StackNavigation Demo

```rust
let stack_nav = self.ui.stack_navigation(cx, ids!(stack_nav_demo));
if self.ui.button(cx, ids!(push_view_a)).clicked(&actions) {
    stack_nav.push(cx, live_id!(stack_view_a));
}
```

完整演示 StackNavigation 的 push/pop/pop_to_root 操作。

### Responsive Nav Demo

```rust
// Desktop
for (item_id, title) in desktop_items {
    if let Some(_) = self.ui.view(cx, item_id).finger_down(actions) {
        // 桌面端：列表和详情并排显示
    }
}
// Mobile
for (item_id, title) in mobile_items {
    if let Some(_) = self.ui.view(cx, item_id).finger_down(actions) {
        mobile_nav.push(cx, live_id!(mobile_detail_view));
    }
}
```

演示响应式导航：桌面端侧边列表 + 详情并排，移动端使用 StackNavigation push。

## script_mod 注册顺序

```rust
impl AppMain for App {
    fn script_mod(vm: &mut ScriptVm) -> ScriptValue {
        crate::makepad_widgets::script_mod(vm);
        crate::layout_templates::script_mod(vm);
        crate::demofiletree::script_mod(vm);
        // ... 所有 tab_* 模块 ...
        self::script_mod(vm)
    }
}
```

注册顺序严格依赖：先基础 widgets，再布局模板，再各 Demo 模块，最后是 app.rs 自身的 `script_mod`。

```rust
fn handle_event(&mut self, cx: &mut Cx, event: &Event) {
    self.match_event(cx, event);
    self.ui.handle_event(cx, event, &mut Scope::empty());
}
```

事件处理双向路径：`match_event` 处理上层逻辑 → `ui.handle_event` 向下分发到各组件。
