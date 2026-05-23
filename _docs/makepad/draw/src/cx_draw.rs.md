# `draw/src/cx_draw.rs` — CxDraw：绘图核心上下文

## 辅助类型

### `PassStackItem`

```rust
pub struct PassStackItem {
    pub pass_id: DrawPassId,
    dpi_factor: f64,
    draw_list_stack_len: usize,
}
```

记录一个 DrawPass 入栈时的快照：pass ID、DPI 缩放因子、以及入栈时刻 `draw_list_stack` 的长度。`end_pass()` 用此长度做完整性检测，确保 begin/end 正确配对。

## `CxDraw` 结构体

```rust
pub struct CxDraw<'a> {
    pub cx: &'a mut Cx,
    pub draw_event: &'a DrawEvent,
    pub(crate) pass_stack: Vec<PassStackItem>,
    pub draw_list_stack: Vec<DrawListId>,
    pub fonts: Rc<RefCell<Fonts>>,
    pub nav_tree_rc: CxNavTreeRc,
    pub rustybuzz_buffer: Option<UnicodeBuffer>,
}
```

### 职责划分

CxDraw 是所有 2D/3D 绘制的**核心中间层**，位于 `Cx`（平台抽象）之上、`Cx2d` 之下：

| 字段 | 职责 |
|------|------|
| `cx: &mut Cx` | 底层平台上下文（窗口、pass、draw list 的实际存储） |
| `draw_event: &DrawEvent` | 当前帧的绘制事件（含时间戳） |
| `pass_stack` | DrawPass 堆栈，入栈时记录快照，用于 pass 嵌套管理 |
| `draw_list_stack` | DrawList 堆栈，`begin_view/end_view` 操作此栈 |
| `fonts` | 共享字体系统（`Rc<RefCell>` 实现全局单例） |
| `nav_tree_rc` | 导航树（`Rc<RefCell>` 实现全局单例），用于焦点遍历 |
| `rustybuzz_buffer` | 文本 shaping 的 Unicode 缓冲区（`Option` 表示延迟初始化） |

### `Deref`/`DerefMut` 到 `Cx`

```rust
impl<'a> Deref for CxDraw<'a> {
    type Target = Cx;
    fn deref(&self) -> &Self::Target { self.cx }
}
```

CxDraw 自动解引用到 `Cx`，可以直接调用 `self.passes[id]`、`self.draw_lists[id]` 等底层方法。

## `Drop` 实现

```rust
impl<'a> Drop for CxDraw<'a> {
    fn drop(&mut self) {
        if !self.fonts.borrow_mut().prepare_textures(self.cx) {
            self.cx.redraw_all();
        }
    }
}
```

### 实现逻辑

1. CxDraw 析构时调用 `fonts.prepare_textures(self.cx)`，将本次帧中排版/光栅化产生的 glyph 上传到 GPU 纹理
2. 如果上传失败（`prepare_textures` 返回 `false`），说明字体图集已变更，需要 `redraw_all()` 触发所有窗口重绘
3. 这是**懒渲染管线**的关键环节：glyph 在绘制过程中被按需光栅化，帧结束时批量上传

## 构造函数

### `new(cx, draw_event)`

```rust
pub fn new(cx: &'a mut Cx, draw_event: &'a DrawEvent) -> Self
```

#### 实现逻辑

1. 调用 `lazy_construct_fonts(cx)` —— 确保 `Fonts` 全局单例已初始化
2. 调用 `lazy_construct_nav_tree(cx)` —— 确保 `CxNavTree` 全局单例已初始化
3. `cx.redraw_id += 1` —— 每次帧开始递增重绘 ID，用于 dirty tracking
4. 从 `cx` 的全局存储中取出字体和导航树的 `Rc<RefCell>` 引用
5. 调用 `fonts.prepare_atlases_if_needed(cx)` 确保图集已准备
6. 初始化 `rustybuzz_buffer: Some(UnicodeBuffer::new())`，供文本 shaping 使用

## 基本查询方法

### `time()` — 获取当前绘制事件的时间戳

```rust
pub fn time(&self) -> f64 { self.draw_event.time }
```

用于动画循环，返回 `DrawEvent.time`，即当前帧的精确时间。

### `lazy_construct_fonts(cx)` — 延迟初始化字体系统

```rust
pub fn lazy_construct_fonts(cx: &mut Cx) -> bool
```

1. 检查 `cx.has_global::<Rc<RefCell<Fonts>>>()`，如果已有则返回 `false`
2. 否则创建 `Fonts::new(cx, layouter::Settings::default())`，并定义一个空的默认字体族（index = 0）
3. 通过 `cx.set_global()` 注册为全局单例

### `get_current_window_id()` — 获取当前 pass 所属窗口

```rust
pub fn get_current_window_id(&self) -> Option<WindowId>
```

通过 `pass_stack.last()` 取出当前 pass ID，委托 `Cx` 查询其所属窗口。

### `current_dpi_factor()` — 当前 DPI 缩放因子

```rust
pub fn current_dpi_factor(&self) -> f64
```

直接从 `pass_stack.last().unwrap().dpi_factor` 返回。

### `set_current_pass_dpi_factor(dpi_factor)` — 设置当前 pass 的 DPI

```rust
pub fn set_current_pass_dpi_factor(&mut self, dpi_factor: f64)
```

1. 更新栈顶 `PassStackItem.dpi_factor`
2. 同步更新 `self.passes[pass_id].dpi_factor` 和底层 `CxPass.set_dpi_factor()`

### `inside_pass()` — 检查是否已进入 pass

```rust
pub fn inside_pass(&self) -> bool { !self.pass_stack.is_empty() }
```

如果 pass 栈不为空则返回 `true`。

## DrawPass 管理

### `make_child_pass(pass)` — 将 pass 标记为其他 pass 的子 pass

```rust
pub fn make_child_pass(&mut self, pass: &DrawPass)
```

取出栈顶 pass 的 ID，将新 pass 的 `parent` 设为 `DrawPass(parent_id)`。

### `begin_pass(pass, dpi_override)` — 开始一个 DrawPass

```rust
pub fn begin_pass(&mut self, pass: &DrawPass, dpi_override: Option<f64>)
```

#### 实现逻辑

1. 根据 DPI 参数决定是使用 `dpi_override` 还是从父级/窗口继承：
   - `dpi_override = Some(f)` → 直接使用
   - parent 是 `Window(window_id)` → 从窗口获取 inner_size 设置 pass_rect，再委托 `get_delegated_dpi_factor`
   - parent 是 `DrawPass(pass_id)` → 从父 pass 继承 pass_rect 和 DPI
2. 将 DPI 因子的计算结果存入 `passes[pass_id].dpi_factor`
3. 将 `PassStackItem`（pass_id、dpi_factor、当前 draw_list_stack 长度）推入 `pass_stack`
4. `main_draw_list_id` 初始设为 `None`，待 `begin_maybe` 中设置

### `end_pass(pass)` — 结束一个 DrawPass

```rust
pub fn end_pass(&mut self, pass: &DrawPass)
```

#### 实现逻辑

1. 从栈顶弹出 `PassStackItem`，panic 校验 pass_id 一致性
2. 检查 `draw_list_stack` 的长度恢复为入栈时的值，否则 panic 提示缺少 `end_view`
3. （注释中保留了对 Turtle 栈长度的校验，当前已禁用）

### `set_pass_area(pass, area)` — 设置 pass 的可见区域

```rust
pub fn set_pass_area(&mut self, pass: &DrawPass, area: Area)
```

将 pass 的 `pass_rect` 设为 `CxDrawPassRect::Area(area)`。

### `set_pass_area_with_origin(pass, area, origin)` — 设置带偏移的区域

```rust
pub fn set_pass_area_with_origin(&mut self, pass: &DrawPass, area: Area, origin: Vec2d)
```

将 pass 的 `pass_rect` 设为 `CxDrawPassRect::AreaOrigin(area, origin)`，其中 `origin` 表示偏移量。

### `set_pass_shift_scale(pass, shift, scale)` — 设置视口平移与缩放

```rust
pub fn set_pass_shift_scale(&mut self, pass: &DrawPass, shift: Vec2d, scale: Vec2d)
```

设置 pass 的 `view_shift` 和 `view_scale`，用于 pass 内部的坐标变换（例如 3D 场景的相机控制）。

### `current_pass_size()` — 获取当前 pass 的尺寸

```rust
pub fn current_pass_size(&self) -> Vec2d
```

1. 根据栈顶 pass_id 调用 `cx.get_pass_rect()` 获取矩形
2. 若成功返回 `.size`，失败返回 `dvec2(0.0, 0.0)`

### `append_sub_draw_list(draw_list_2d)` — 将子 DrawList 追加到当前 DrawList

```rust
pub fn append_sub_draw_list(&mut self, draw_list_2d: &DrawList2d)
```

通过 `self.cx.draw_lists[current_parent].append_sub_list()` 将 `draw_list_2d` 注册为当前 draw list 的子列表。这构建了 DrawList 的树形层次结构。
