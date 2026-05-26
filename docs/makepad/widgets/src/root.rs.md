# root.rs — Root 顶层组件与组件映射

## 文件概述

`Root` 是 Makepad 应用最顶层的 Widget 组件。它是应用入口，负责：

1. 初始化 UI 开始循环
2. 管理组件映射表（`widget_map`）— 脚本类型到 Rust 结构体的注册
3. 管理启动声明周期（`on_startup`）
4. 作为 `WidgetTree` 的根容器

---

## `Root` 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct Root {
    #[source] source: ScriptObjectRef,
    #[walk] walk: Walk,
    #[layout] layout: Layout,
    #[redraw] #[live] draw_bg: DrawQuad,
    #[live] widget_map: WidgetMap,         // 组件映射表
    #[live] start_node: Option<WidgetRef>, // 启动后挂载的子 Widget
    #[rust] widget_tree: WidgetTree,       // Widget 图结构
    #[rust] is_started: bool,             // 是否已启动
}
```

---

## WidgetMap — 组件映射

```rust
pub struct WidgetMap {
    map: HashMap<LiveId, LiveId>,  // 脚本类型名 → Rust 结构体类型名
}
```

`WidgetMap` 维护了从脚本类型标识符到 Rust 结构体类型的映射关系。注册方式：

### 脚本 DSL 注册

在 `live_design!` 或 `script_mod!` 中自动注册：

```
// 脚本 DSL 中 widget_map 自动填充
mod.widgets.MyButton = set_type_default() do mod.widgets.Button {
    // 此处设置 MyButton 的默认属性
}
```

### Rust 层注册

```rust
// 在 Root.on_startup 或 Widget 初始化时
let widget_map = cx.widget_map();
widget_map.register("MyButton", "Button");
```

注册后，当每次创建该类型的 Widget 时，系统通过 `widget_map` 找到对应的 Rust 结构体进行实例化。

### 组件查找

```rust
impl WidgetMap {
    pub fn find(&self, type_id: LiveId) -> Option<&LiveId> {
        self.map.get(&type_id)
    }
}
```

用于 `WidgetRef` 的 `from_resource()` 方法：当从资源文件加载 widget 树时，通过 `widget_map` 将脚本中的类型名称解析为实际的 Rust 结构体。

---

## 启动生命周期

```rust
impl Widget for Root {
    fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
        // 1. 首次绘制时执行启动初始化
        if !self.is_started {
            self.is_started = true;
            self.on_startup(cx, scope);
        }
        
        // 2. 开始 WidgetTree 帧
        cx.begin_widget_tree(&mut self.widget_tree);
        
        // 3. 绘制 start_node（即应用主界面）
        if let Some(ref mut node) = self.start_node {
            node.draw_walk(cx, scope, walk);
        }
        
        // 4. 结束 WidgetTree 帧
        cx.end_widget_tree(&mut self.widget_tree);
        DrawStep::Done(self.area)
    }
}
```

### `draw_walk` 流程

| 步骤 | 动作 | 细节 |
|------|------|------|
| 1 | 检查 is_started | 未初始化则调用 on_startup |
| 2 | begin_widget_tree | 重置帧索引，准备 Area 注册 |
| 3 | 绘制 start_node | 递归绘制整个 UI 树 |
| 4 | end_widget_tree | 清理过期条目，更新查询索引 |
| 5 | 返回 Done | 提供根 Area |

### `on_startup`

```rust
fn on_startup(&mut self, cx, scope) {
    // 1. 触发所有子 Widget 的 on_startup 回调
    // 2. 注册组件映射
    // 3. 执行应用级初始化
}
```

- 在绘制树之前执行，确保所有组件已就绪。
- 通过 `scope` 向子组件传播启动信号。
- 如果子组件也实现了 `on_startup`，递归调用。

---

## 事件处理

```rust
fn handle_event(&mut self, cx, event, scope) {
    // 1. self.match_event(cx, event) — 应用级事件
    // 2. self.ui.handle_event(cx, event, scope) — UI 树事件
    // 3. 通过 widget_tree 进行命中测试和事件路由
}
```

Root 的事件处理分两层：

1. **应用级**：通过 `match_event` 处理窗口事件、关闭请求、键盘快捷键等。
2. **UI 级**：通过 `self.ui.handle_event` 将事件传递给子组件树。

WidgetTree 在 `handle_event` 中用于从屏幕坐标定位目标 Widget。Root 持有 `WidgetTree` 的唯一引用，负责帧的 begin/end。

---

## 与 App 的关系

典型的 App 结构：

```rust
impl App {
    fn run(vm: &mut ScriptVm) -> Self {
        // ...
        App::from_script_mod(vm, self::script_mod)
    }
}

impl AppMain for App {
    fn handle_event(&mut self, cx, event) {
        self.match_event(cx, event);
        self.ui.handle_event(cx, event, &mut Scope::empty());
    }
}
```

- `App` 使用 `Root` 作为 UI 树的根节点。
- App 的 `handle_event` 调用 Root 的 `handle_event`，从而启动整个事件分发流程。
- Root 负责 WidgetTree 生命周期管理，App 不需要关心底层实现。
- `scope` 传递在 App 层被简化为 `Scope::empty()`，真正的 scope 深度由 Root→Window→View 链管理。
