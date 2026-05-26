# window.rs — Window 窗口组件

## 文件概述

`Window` 是 Makepad 应用的主窗口 O 组件。它包装了底层平台窗口（通过 `window` 模块），提供标题栏、缩放按钮、键盘导航、高斯背景模糊等功能。

---

## `Window` 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct Window {
    #[source] source: ScriptObjectRef,
    #[walk] walk: Walk,
    #[layout] layout: Layout,
    #[redraw] #[live] draw_bg: DrawQuad,
    #[redraw] #[live] draw_titlebar: DrawQuad,        // 标题栏背景
    #[live] animator: Animator,
    #[live] window: live_lib::Window,                   // 底层平台窗口
    #[live] title: String,                              // 窗口标题
    #[live] titlebar: bool,                             // 是否显示标题栏
    #[live] titlebar_buttons: bool,                     // 显示缩放/关闭按钮
    #[live] resizable: bool,                            // 是否可调整大小
    #[live] background_blur: Option<BackgroundBlur>,   // 高斯模糊配置
    #[live] fullscreen: Option<FullScreen>,             // 全屏模式
    #[live] window_menu: Option<WindowMenu>,           // 窗口菜单栏
    #[rust] area: Area,
    #[rust] content_area: Area,                         // 内容区域（标题栏下方）
}
```

---

## 平台窗口管理

### 窗口创建

```rust
fn on_startup(&mut self, cx, scope) {
    // 1. 调用 cx.create_window() 创建平台窗口
    // 2. 设置窗口属性：标题、尺寸、可缩放
    // 3. 注册窗口事件回调
}
```

`live_lib::Window` 是与平台窗口绑定的结构体，负责：

- 窗口大小/位置管理
- 窗口关闭/最小化/全屏操作
- DPI 缩放处理
- 平台事件转发

### 窗口属性同步

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    // 1. 同步 live 属性到底层窗口
    self.window.set_title(&self.title);
    self.window.set_resizable(self.resizable);
    
    // 2. 检查全屏状态变化
    if let Some(fs) = &self.fullscreen {
        self.window.set_fullscreen(*fs);
    }
    
    // 3. 应用背景模糊效果
    if let Some(blur) = &self.background_blur {
        self.apply_background_blur(cx, blur);
    }
    
    // 4. 绘制窗口内容
}
```

---

## 标题栏与按钮

### 标题栏布局

当 `titlebar: true` 时，窗口顶部显示标题栏：

```
┌─────────────────────────────────────┐
│ ← 窗口标题   [─] [□] [✕] │  ← titlebar
│                                     │
│  body content (内容区域)            │
│                                     │
└─────────────────────────────────────┘
```

- 标题栏占据窗口顶部固定高度（通常 ~40px）。
- 标题文字居中或左对齐。
- 窗口按钮在右上方（macOS 风格在左上方）。

### 窗口按钮

当 `titlebar_buttons: true` 时，显示三个窗口控制按钮：

- **关闭**（✕）：发出 `CloseEvent`，关闭窗口。
- **最小化**（─）：`window.set_minimized(true)`。
- **最大化/还原**（□）：在 `fullscreen` 和普通状态间切换。

按钮交互：

```rust
fn handle_titlebar_click(&mut self, cx, mouse_pos) {
    if close_btn_rect.contains(mouse_pos) {
        cx.emit(Event::Close);
    } else if minimize_btn_rect.contains(mouse_pos) {
        self.window.set_minimized(true);
    } else if maximize_btn_rect.contains(mouse_pos) {
        self.window.toggle_fullscreen();
    }
}
```

### 标题栏拖拽

鼠标在标题栏上按下时触发窗口移动：

```rust
fn handle_titlebar_drag(&mut self, cx, mouse_pos) {
    if self.titlebar_hit {
        let delta = mouse_pos - self.drag_start_pos;
        self.window.set_position(self.window.position() + delta);
    }
}
```

---

## 背景高斯模糊

```rust
pub struct BackgroundBlur {
    pub radius: f64,        // 模糊半径
    pub enabled: bool,
}
```

`background_blur` 是可选字段，配置窗口背景的高斯模糊效果：

1. 检查平台是否支持背景模糊（macOS > 10.14 支持 `NSVisualEffectView`）。
2. 设置平台窗口的 `background_blur` 属性。
3. 半透明窗口区域会自动显示模糊效果。

### 平台适配

- **macOS**：使用 `NSVisualEffectView` 系统实现，支持 vibrancy 模式。
- **Windows**：使用 `SetWindowCompositionAttribute` 或 DirectComposition 实现。
- **Linux**：通过 KDE blur 或 GNOME shell 扩展支持。

---

## 键盘导航

Window 级别处理键盘导航键：

| 按键 | 行为 |
|------|------|
| Tab | 焦点移动到下一个可聚焦 Widget |
| Shift+Tab | 焦点移动到上一个可聚焦 Widget |
| Esc | 关闭窗口 / 取消当前操作 |
| Enter | 触发选中按钮的 action |
| Arrow keys | 导航列表/菜单项 |

Tab 顺序由 WidgetTree 中的 Area 注册顺序决定。Window 维护一个焦点 Widget 引用，当 Tab 被按下时按照注册顺序推进焦点。

---

## 全屏模式

`fullscreen: Option<FullScreen>` 支持：

- `None`：普通窗口模式
- `Some(FullScreen::Windowed)`：无边框全屏
- `Some(FullScreen::Exclusive)`：独占全屏（可能改变显示分辨率）

切换全屏时自动调整窗口尺寸和渲染区域。

---

## 内容区域

`content_area` 标识标题栏下方的可用区域：

```rust
fn draw_walk(&mut self, cx, scope, walk) -> DrawStep {
    cx.begin_turtle(walk, self.layout);
    let window_rect = cx.turtle().rect();
    
    if self.titlebar {
        // 绘制标题栏
        let titlebar_rect = Rect::new(
            window_rect.pos,
            DVec2::new(window_rect.size.x, TITLEBAR_HEIGHT)
        );
        self.draw_titlebar.draw_abs(cx, titlebar_rect);
        
        // 内容区域偏移到标题栏下方
        let content_rect = Rect::new(
            DVec2::new(window_rect.pos.x, window_rect.pos.y + TITLEBAR_HEIGHT),
            DVec2::new(window_rect.size.x, window_rect.size.y - TITLEBAR_HEIGHT)
        );
        self.content_area = cx.turtle().add_rect(content_rect);
    }
    
    // 绘制 body 子组件
    cx.end_turtle_with_area(&mut self.area);
    DrawStep::Done(self.area)
}
```

`content_area` 用于事件路由：点击标题栏的事件由 Window 处理，内容区域的事件传递给子组件。
