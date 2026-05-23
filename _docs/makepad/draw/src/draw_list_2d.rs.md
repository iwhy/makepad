# `draw/src/draw_list_2d.rs` — DrawList2d：2D 绘制列表上层 API

## 文件结构概览

此文件包含四大逻辑块：

1. **`DrawListExt` trait** — 为 `DrawList` 添加高级操作（变换、begin/end、点映射、调试、重绘）
2. **`DrawList2d` 结构体** — 对 `DrawList` 的轻量封装，加入 dirty 矩形检测和 overlay 管理
3. **`CxDraw` 的 instance 管理方法** — 单实例 / 批量实例的添加
4. **`Cx2d` 的对齐实例方法** — 在 Cx2d 层面添加对齐支持
5. **`Redrawing` 类型** — `Result<(), ()>` 的别名，表示是否正在重绘

---

## `DrawListExt` trait

为 `DrawList` 增加了一组高级操作方法。

### `draw_list_id()` — 返回自身的 `DrawListId`

```rust
fn draw_list_id(&self) -> DrawListId { self.id() }
```

### `set_view_transform(cx, mat)` — 递归设置视图变换矩阵

```rust
fn set_view_transform(&self, cx: &mut Cx, mat: &Mat4f)
```

#### 实现逻辑

1. 递归遍历 draw list 中的所有 draw item
2. 对当前 draw list 设置 `view_transform = *mat`
3. 遍历所有 draw item，对每个是 `sub_list` 的 item 递归调用自身
4. 这使得变换矩阵能**向下传播**到所有子 draw list

### `set_view_transform_self_only(cx, mat)` — 仅设置自身的变换矩阵

```rust
fn set_view_transform_self_only(&self, cx: &mut Cx, mat: &Mat4f)
```

只修改当前 draw list 的 `view_transform`，不递归到子级。用于独立变换某个 draw list 而不影响子树。

### `begin_always(cx)` — 无条件开始绘制

```rust
fn begin_always(&mut self, cx: &mut CxDraw) {
    self.begin_maybe(cx, true).expect_redraw();
}
```

总是调用 `begin_maybe(cx, true)`，即无论是否需要重绘都执行 begin。`expect_redraw()` 在返回非重绘时 panic。

### `begin_maybe(cx, will_redraw)` — 有条件开始绘制

```rust
fn begin_maybe(&mut self, cx: &mut CxDraw, will_redraw: bool) -> Redrawing
```

#### 实现逻辑（核心方法）

1. **获取 pass 上下文**：`pass_id = cx.pass_stack.last().unwrap().pass_id`，将当前 draw list 关联到此 pass
2. **记录 codeflow parent**：`codeflow_parent_id = cx.draw_list_stack.last()`，用于增量重绘扫描
3. **main_draw_list 注册**：如果当前 pass 还没有 main draw list，则将此 draw list 注册为 main
4. **父子 list 关联**：如果有父级且自身不是 main，则：
   - `parent.append_sub_list(redraw_id, self.id())`
   - `cx.nav_list_item_push(parent_id, NavItem::Child(self.id()))` 将子关系记录到导航树
5. **增量重绘判断**：
   - 如果 draw list 为空 或 `will_redraw == true` → 进入绘制路径
   - 否则返回 `Redrawing::no()`（跳过本帧的绘制构建）
6. **标记 paint_dirty**：如果是 main draw list，设置 `pass.paint_dirty = true`
7. **清空 draw items**：`self.clear_draw_items(redraw_id)`——清除旧帧的绘制数据
8. **清空导航列表**：`cx.nav_list_clear(self.id())`
9. **推入 draw list 栈**：`cx.draw_list_stack.push(self.id())`

### `end(cx)` — 结束绘制

```rust
fn end(&mut self, cx: &mut CxDraw)
```

1. 从 `draw_list_stack` 弹出，panic 校验 ID 一致性
2. 校验 `redraw_id` 是否匹配（确保 begin 和 end 在同一重绘周期）

### `get_view_transform(cx)` — 获取当前视图变换矩阵

```rust
fn get_view_transform(&self, cx: &Cx) -> Mat4f
```

从 `draw_list_uniforms.view_transform` 读取。

### `map_point_to_local(cx, world)` — 世界坐标 → 本地坐标

```rust
fn map_point_to_local(&self, cx: &Cx, world: DVec2) -> DVec2
```

1. 对视图变换矩阵求逆
2. `inverse.transform_vec4()` 执行变换
3. 进行齐次除法（除以 `w`），处理透视投影

### `map_point_from_local(cx, local)` — 本地坐标 → 世界坐标

```rust
fn map_point_from_local(&self, cx: &Cx, local: DVec2) -> DVec2
```

直接用变换矩阵乘以坐标向量，同样处理齐次除法。

### 调试方法

- `debug_parent_draw_list_id(cx)` → 返回 `codeflow_parent_id`
- `debug_child_draw_list_ids(cx)` → 遍历 draw item，收集所有 `sub_list` 的 ID

### 重绘方法

- `redraw(cx)` → `cx.redraw_list(self.id())`，标记此 draw list 重绘
- `redraw_self_and_children(cx)` → `cx.redraw_list_and_children(self.id())`，递归标记

---

## `DrawList2d` 结构体

```rust
pub struct DrawList2d {
    pub(crate) draw_list: DrawList,
    pub(crate) dirty_check_rect: Rect,
    overlay_active: bool,
}
```

### `Deref`/`DerefMut` 到 `DrawList`

```rust
impl Deref for DrawList2d { type Target = DrawList; }
```

可直接调用 `DrawList` 的方法（如 `id()`、`clear_draw_items()` 等）。

### Script 声明

实现了 `ScriptHook`、`ScriptApply`、`ScriptNew`，使其可在 `script_mod!` 中创建。

### `new(cx)` — 构造函数

```rust
pub fn new(cx: &mut Cx) -> Self
```

1. `cx.draw_lists.alloc()` 分配一个新的 `DrawList`，获得唯一 ID
2. `dirty_check_rect` 初始化为 `Default::default()`（零矩形）
3. `overlay_active` 初始为 `false`

### `begin(cx, walk)` — 结合脏矩形检测开始绘制

```rust
pub fn begin(&mut self, cx: &mut Cx2d, walk: Walk) -> Redrawing
```

1. `cx.will_redraw(self, walk)` 根据 walk 检测位置/大小是否变化
2. 委托 `self.begin_maybe(cx, will_redraw)` 执行实际的 begin 逻辑

### `end(cx)` — 结束，清理 overlay 状态

```rust
pub fn end(&mut self, cx: &mut Cx2d)
```

1. 如果 `overlay_active == true`，将其设为 `false` 并减少 `cx.overlay_draw_depth`
2. 委托 `self.draw_list.end(cx)` 执行 DrawList 层的 end

### Overlay 方法

#### `begin_overlay_last(cx)` / `begin_overlay_reuse(cx)`

两者都委托给 `begin_overlay_inner`：
- `begin_overlay_last(cx)` → `always_last = true`
- `begin_overlay_reuse(cx)` → `always_last = false`

#### `begin_overlay_inner(cx, always_last)` — 覆盖层 begin

```rust
pub fn begin_overlay_inner(&mut self, cx: &mut Cx2d, always_last: bool)
```

#### 实现逻辑

1. 获取 pass_id：优先使用 `cx.overlay_pass_id`，否则使用当前 pass
2. 获取 `codeflow_parent_id = cx.draw_list_stack.last().unwrap()`
3. 获取 `overlay_id = cx.overlay_id.unwrap()`——overlay 的容器 draw list
4. **注册到 overlay 容器**：
   - `always_last == true` → `store_sub_list_last()`（置于绘制顺序末尾，覆盖所有内容）
   - `always_last == false` → `store_sub_list()`（正常位置）
5. 如果 `overlay_active` 尚为 false，标记为 true 并增加 `cx.overlay_draw_depth`
6. 记录 NavItem::Child 到导航树
7. 清空 draw items，清空导航列表，推入 draw list 栈

---

## DrawCall 与 Instance 管理

以下方法在 `CxDraw` 上实现，但定义在 `draw_list_2d.rs` 中。

### `new_draw_call(draw_vars)` / `append_to_draw_call(draw_vars)`

```rust
pub fn new_draw_call(&mut self, draw_vars: &DrawVars) -> Option<&mut CxDrawItem>
pub fn append_to_draw_call(&mut self, draw_vars: &DrawVars) -> Option<&mut CxDrawItem>
```

两者都委托给 `get_draw_call`：
- `new_draw_call` → `append = false`
- `append_to_draw_call` → `append = true`

### `get_draw_call(append, draw_vars)` — 获取或创建 draw call

```rust
pub fn get_draw_call(&mut self, append: bool, draw_vars: &DrawVars) -> Option<&mut CxDrawItem>
```

#### 实现逻辑

1. 如果 `draw_vars.draw_shader_id` 为 None，返回 None（无 shader 时不创建 draw call）
2. 获取当前 DrawList 的 ID
3. 如果 `append == true` 且 shader 没有 `draw_call_always` 标志：
   - 尝试通过 `draw_list.find_appendable_drawcall(sh, draw_vars)` 查找可合并的 draw call
   - 找到则直接返回该 draw item，实现 **draw call 合并**（同 shader、同状态的连续绘制合并为一次 GPU draw call）
4. 未找到可合并项或不应合并时，调用 `draw_list.append_draw_call(redraw_id, sh, draw_vars)` 创建新的 draw call

### `begin_many_instances(draw_vars)` — 开始批量实例

```rust
pub fn begin_many_instances(&mut self, draw_vars: &DrawVars) -> Option<ManyInstances>
```

#### 实现逻辑

1. 调用 `append_to_draw_call` 获取或创建 draw item
2. 如果 draw item 不存在（shader 无效）则返回 None
3. 将 draw item 中现有的 `instances` 向量取出（swap 到局部变量）
4. 构造 `ManyInstances`：
   - `instance_area`：记录 draw_list_id、draw_item_id、instance_offset（当前长度）和 redraw_id
   - `instances`：取出的实例数据 Vec\<f32\>
   - `aligned`：暂为 None（Cx2d 层会设置）

### `end_many_instances(many_instances)` — 结束批量实例

```rust
pub fn end_many_instances(&mut self, many_instances: ManyInstances) -> Area
```

1. 将 `many_instances.instances` swap 回 draw item
2. 计算实际添加的实例数：`(新长度 - 旧偏移) / total_instance_slots`
3. 将 `InstanceArea` 转换为 `Area` 返回，供点击检测使用

### `add_instance(draw_vars)` — 添加单一实例

```rust
pub fn add_instance(&mut self, draw_vars: &DrawVars) -> Area
```

#### 实现逻辑

1. 将 `draw_vars.as_slice()` 作为实例数据
2. 调用 `append_to_draw_call` 获取 draw item（如果 shader 无效则返回 Area::Empty）
3. 校验数据长度是 `total_instance_slots` 的整数倍
4. 记录 `InstanceArea`（draw_list_id、draw_item_id、instance_count、offset、redraw_id）
5. 将数据 `extend_from_slice` 写入 draw item 的 instances 向量
6. 返回对应的 Area

---

## Cx2d 的对齐实例方法

### `begin_many_aligned_instances(draw_vars)` — 带对齐支持的批量实例开始

```rust
pub fn begin_many_aligned_instances(&mut self, draw_vars: &DrawVars) -> Option<ManyInstances>
```

1. 调用 `begin_many_instances` 获取基础 `ManyInstances`
2. 如果成功，在 `self.align_list` 预留一个 `AlignEntry::Unset` 位置
3. 将对齐索引写入 `many_instances.aligned`

### `end_many_instances(many_instances)` — 结束带对齐的批量实例（Cx2d 重载）

```rust
pub fn end_many_instances(&mut self, many_instances: ManyInstances) -> Area
```

与 CxDraw 版本相同，额外在 `aligned` 有值时将 `Area` 写入 `align_list[aligned]`。

### `add_aligned_instance(draw_vars)` — 添加带对齐的单一实例

```rust
pub fn add_aligned_instance(&mut self, draw_vars: &DrawVars) -> Area
```

1. 执行 `add_instance` 相同的逻辑
2. 额外将返回的 `Area` 推入 `self.align_list`

### `add_aligned_rect_area(area, rect)` — 添加对齐矩形区域

```rust
pub fn add_aligned_rect_area(&mut self, area: &mut Area, rect: Rect)
```

1. 在 `draw_list.rect_areas` 中分配一个新的 `CxRectArea`
2. 构造基于 `RectArea` 的 Area
3. 推入 `align_list`
4. 调用 `update_area_refs` 更新外部 area 引用
5. 将传入的 `area` 更新为新区域

---

## 辅助类型

### `ManyInstances`

```rust
pub struct ManyInstances {
    pub instance_area: InstanceArea,
    pub aligned: Option<usize>,
    pub instances: Vec<f32>,
}
```

- `instance_area`：指向 draw item 中实例区间的元数据
- `aligned`：如果是在 Cx2d 下开启的，则包含 align_list 下标
- `instances`：实例数据的实际缓冲区

### `AlignedInstance`

```rust
pub struct AlignedInstance {
    pub inst: InstanceArea,
    pub index: usize,
}
```

### `Redrawing` — `Result<(), ()>` 的类型别名

```rust
pub type Redrawing = Result<(), ()>;
```

### `RedrawingApi` trait

```rust
pub trait RedrawingApi {
    fn no() -> Redrawing { Result::Err(()) }
    fn yes() -> Redrawing { Result::Ok(()) }
    fn is_redrawing(&self) -> bool;
    fn is_not_redrawing(&self) -> bool;
    fn expect_redraw(&self);
}
```

方便的方法：
- `no()` → 不重绘
- `yes()` → 重绘
- `is_redrawing()` → `self.is_ok()`
- `is_not_redrawing()` → `self.is_err()`
- `expect_redraw()` → 如果不是重绘则 panic

---

## 单元测试

```rust
#[cfg(test)]
mod tests {
```

### `self_only_transform_does_not_touch_children`

验证 `set_view_transform_self_only` 不会递归修改子 draw list 的变换矩阵。

1. 创建 parent 和 child 两个 DrawList2d，建立父子关系
2. 分别设置不同的变换矩阵
3. 断言两个 list 的变换矩阵保持独立

### `recursive_transform_still_updates_children`

验证 `set_view_transform`（递归版本）能正确传播到子 draw list。

1. 创建 parent 和 child，建立父子关系
2. 对 parent 调用 `set_view_transform`
3. 断言 child 的变换矩阵与 parent 一致

### `point_mapping_round_trips_translation`

验证平移变换下，`map_point_from_local` 和 `map_point_to_local` 互为逆操作。

1. 创建一个平移 `(10, 20)` 的 draw list
2. `map_point_from_local(5, 6)` → 期望 `(15, 26)`
3. `map_point_to_local(15, 26)` → 期望 `(5, 6)`

### `debug_helpers_report_parent_children_and_transform`

验证调试辅助方法返回正确的父子关系和变换矩阵。
