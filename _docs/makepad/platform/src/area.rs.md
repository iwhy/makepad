# `area.rs` — Area 交互区域与命中测试基础

## 概述

`area.rs` 定义了 Makepad 框架中 UI 交互区域的基础设施。`Area` 枚举是整个框架中"可交互区域"的统一表示，通过它可以在绘制列表（DrawList）中定位具体的实例或矩形区域。

Area 的核心作用：
1. **命中测试**：判断用户点击/触摸落在哪个 widget 上
2. **区域重绘**：通过 Area 标记需要重绘的区域
3. **状态关联**：键盘焦点、IME、拖放、手指触摸等交互状态都通过 Area 关联
4. **坐标转换**：在绝对坐标和相对坐标之间转换

---

## 核心结构体

### `InstanceArea`（第 12-18 行）

```rust
pub struct InstanceArea {
    pub draw_list_id: DrawListId,
    pub draw_item_id: usize,
    pub instance_offset: usize,
    pub instance_count: usize,
    pub redraw_id: u64,
}
```

**字段说明：**
- `draw_list_id`：所属的绘制列表 ID
- `draw_item_id`：绘制列表中的绘制项（DrawItem）索引
- `instance_offset`：实例缓存中的起始偏移（float 槽位），指向 GPU 实例数据的起始位置
- `instance_count`：覆盖的实例数量
- `redraw_id`：创建时的绘制列表重绘 ID，用于判断 Area 是否有效

**实现逻辑：** `InstanceArea` 精确地标识了 GPU 实例缓冲区中的一个连续段。当 widget 在 draw call 中写入实例数据时，系统返回一个 `InstanceArea`，后续可以通过它读取/写入矩形位置、修改颜色等属性。`redraw_id` 用于验证——如果 draw list 被重建，旧 `redraw_id` 的 Area 自动失效。

### `RectArea`（第 26-31 行）

```rust
pub struct RectArea {
    pub draw_list_id: DrawListId,
    pub rect_id: usize,
    pub redraw_id: u64,
}
```

**字段说明：**
- `draw_list_id`：所属绘制列表 ID
- `rect_id`：绘制列表中矩形区域数组的索引
- `redraw_id`：创建时的重绘 ID

**实现逻辑：** `RectArea` 比 `InstanceArea` 更轻量，不关联 GPU 实例数据，只引用绘制列表中记录的矩形区域数组。

---

## `Area` 枚举（第 33-39 行）

```rust
pub enum Area {
    Empty,
    Instance(InstanceArea),
    Rect(RectArea),
}
```

三种变体：
- `Empty`：空区域，表示没有有效关联——常用于默认值或已经失效的区域
- `Instance(InstanceArea)`：指向 GPU 实例缓冲区的子区域，用于着色器驱动的 widget
- `Rect(RectArea)`：指向绘制列表中的矩形区域

**实现逻辑：** 默认值为 `Area::Empty`。`Instance` 是主要的交互区域类型——几乎所有 widget（Button、Label、View 等）在绘图时都会返回 `InstanceArea`。`Rect` 用于不需要实例数据的简单矩形区域。

---

## `Area` 方法详解

### 基础方法

| 方法 | 说明 |
|------|------|
| `area()` | 返回自身克隆 |
| `redraw(cx)` | 通过 `cx.redraw_area(*self)` 触发该区域的重绘 |
| `is_empty()` | 判断是否为 `Area::Empty` |
| `draw_list_id()` | 提取关联的 `DrawListId`（Instance 和 Rect 变体返回 Some，Empty 返回 None） |
| `redraw_id()` | 提取关联的重绘 ID |
| `is_first_instance()` | 判断是否是指定 draw item 的第一个实例（instance_offset == 0） |

### `is_valid()`——有效性验证（第 147-172 行）

**实现逻辑：**
1. 如果是 `Instance` 变体，先检查 `instance_count > 0`，再通过 `cx.draw_lists.checked_index()` 查找 draw list 是否存在，最后验证 `draw_list.redraw_id == inst.redraw_id`
2. 如果是 `Rect` 变体，类似地检查 draw list 存在且 redraw_id 匹配
3. 任一检查失败返回 `false`

这个验证机制确保 Area 不会指向已重建/释放的 GPU 资源。

### `extend_with()`——区域扩展（第 116-145 行）

**实现逻辑：**
- 如果 self 是 `Area::Empty`，直接返回 `new_area`
- 如果新旧都是 `Instance` 变体且 `draw_list_id` / `draw_item_id` / `redraw_id` 都匹配，则合并：保持旧的 `instance_offset`，将 `instance_count` 扩展为两者的和。这是为了在同一 draw call 中扩展选中区域
- 否则直接返回 `new_area`

### `rect()`——获取矩形区域（第 271-317 行）

**实现逻辑（Instance 变体）：**
1. 验证 area 有效（redraw_id 匹配，instance_count > 0）
2. 获取 draw item 和 draw call，从 draw call 中获取 draw shader
3. 从 draw shader 的 mapping 中读取 `rect_pos` 和 `rect_size` 字段的 float 偏移
4. 从实例 float 缓冲区中提取位置和尺寸数据，构造 `Rect { pos, size }`

**实现逻辑（Rect 变体）：**
1. 验证 draw list 的 redraw_id 匹配
2. 从 `draw_list.rect_areas[rect_id].rect` 直接取出预存矩形

### `clipped_rect()`——获取裁剪后的矩形（第 175-269 行）

**实现逻辑：**
- 在 `rect()` 的基础上增加裁剪处理：
  1. 如果 shader mapping 中有 `draw_clip` 字段，读取 4 个 float 值作为裁剪矩形 `(p1, p2)`
  2. 如果 draw list 有 clip（`draw_list_has_clip`），额外应用 viewport 级别的裁剪 `(p3, p4)`
  3. 如果 draw list 有 `view_shift`，应用平移
  4. 应用链式裁剪：`Rect::clip((p1, p2)).translate(shift).clip((p3, p4))`

### `abs_to_rel()`——绝对坐标转相对坐标（第 319-355 行）

**实现逻辑（Instance 变体）：**
1. 验证 area 有效
2. 读取实例缓冲区中 `rect_pos` 位置的前两个 float 作为 (x, y) 起点
3. 返回 `Vec2d { x: abs.x - x, y: abs.y - y }`

**实现逻辑（Rect 变体）：**
1. 从 `rect_area.rect.pos` 获取起点
2. 返回绝对坐标减去起点

### `set_rect()`——设置矩形位置（第 357-412 行）

**实现逻辑（Instance 变体）：**
1. 验证 area 有效
2. 获取 draw shader mapping 中的 `rect_pos` 和 `rect_size` 偏移
3. 将 `rect.pos.x` / `rect.pos.y` 写入实例缓冲区对应位置
4. 将 `rect.size.x` / `rect.size.y` 写入尺寸位置
5. 有边界检查保护

**实现逻辑（Rect 变体）：**
- 直接更新 `draw_list.rect_areas[rect_id].rect`

---

## 被注释掉的 `get_read_ref` / `get_write_ref`（第 413-515 行）

这两个方法提供了通过 Area 直接读取/写入 GPU 实例数据的通用接口。通过 `LiveId` 查找 shader mapping 中的输入属性位置，返回一个包含 `repeat`、`stride`、`buffer` 的引用。DrawCall uniforms 和 instances 都可以通过这种方式访问。当前版本中已注释掉，可能已被更高层次的抽象替代。

---

## 总结

Area 是 Makepad 渲染系统中连接 CPU 端 widget 逻辑和 GPU 端实例数据的桥梁。它通过 `DrawListId` + `DrawItemId` + `instance_offset` 三级索引精确定位 GPU 实例缓冲区中的子区域。`redraw_id` 生成式验证机制确保了 Area 不会被 dangling 使用——每次 draw list 重建时递增 redraw_id，旧 area 自动失效。
