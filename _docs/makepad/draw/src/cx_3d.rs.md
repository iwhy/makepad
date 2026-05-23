# `cx_3d.rs` — 3D 绘图上下文

**文件路径**: `draw/src/cx_3d.rs` (112 行)  
**核心作用**: 提供 2D 绘图上下文到 3D 场景的桥梁，管理当前 3D 场景状态、世界变换和绘制锚点注册。

---

## 一、`Cx3d` 结构体（第 11–14 行）

```rust
pub struct Cx3d<'a, 'b> {
    pub cx: &'b mut CxDraw<'a>,
    scene_3d: Cx3dState,
}
```

**设计意图**: `Cx3d` 同时包装了 `CxDraw`（2D 绘图上下文）和 `Cx3dState`（3D 场景状态）。通过实现 `Deref<Target = CxDraw>` 和 `DerefMut`（第 16–26 行），可以透明地访问所有 2D 绘图方法，同时额外提供 3D 场景管理功能。

**生命周期参数**:
- `'a`: `CxDraw` 内部引用的生命周期
- `'b`: `CxDraw` 本身的可变借用生命周期

---

## 二、3D 场景生命周期管理

### `new(cx: &'b mut CxDraw<'a>) -> Self`（第 29–34 行）

构造 `Cx3d`，初始化 `scene_3d` 为 `Cx3dState::default()`（场景作用域为 `None`）。

### `begin_scene_3d(&mut self, scene: SceneState3D)`（第 44–46 行）

开始一个 3D 场景的绘制。将传入的 `SceneState3D`（包含时间、相机位置、视图矩阵、投影矩阵、视口矩形）包装为 `SceneScope3D` 并存入 `scene_3d.scene_scope`。

实现逻辑：直接覆盖 `scene_scope` 字段为 `Some(SceneScope3D::new(scene))`，任何之前存在的场景状态被丢弃。

### `end_scene_3d(&mut self)`（第 48–50 行）

结束当前 3D 场景。将 `scene_scope` 设为 `None`，清理场景状态。

---

## 三、场景状态查询

### `scene_3d() -> Option<&SceneScope3D>`（第 36–38 行）

返回当前场景作用域的不可变引用。通过 `Option` 表达"当前是否在 3D 场景中"。

### `scene_3d_mut() -> Option<&mut SceneScope3D>`（第 40–42 行）

返回当前场景作用域的可变引用。

### `scene_state_3d() -> Option<SceneState3D>`（第 52–54 行）

提取 `SceneState3D` 的副本（`Copy` 类型）。映射 `scene_3d()` 到 `scope.scene`，返回 `Option<SceneState3D>`。

---

## 四、世界变换管理

### `scene_world_transform_3d() -> Mat4f`（第 56–60 行）

返回当前场景的世界变换矩阵。如果不在场景中，返回单位矩阵 `Mat4f::identity()`。

### `set_scene_world_transform_3d(&mut self, world_transform: Mat4f) -> Option<Mat4f>`（第 62–67 行）

设置当前场景的世界变换矩阵。返回设置前的旧变换矩阵（包装在 `Option` 中），如果不在场景中则返回 `None`。

实现逻辑：通过 `scene_3d_mut()?` 提前返回，然后 `mem::replace` 风格更新 `world_transform` 字段。

---

## 五、Draw Call 锚点系统

Draw Call 锚点是 3D 场景中标记关键位置的"地标"，用于 2D 覆盖层（如标签、工具提示）与 3D 对象的空间关联。

### `scene_draw_call_anchors_3d() -> Option<&[SceneDrawCallAnchor]>`（第 69–72 行）

返回当前场景所有已注册的 draw call 锚点的切片引用。

### `clear_scene_draw_call_anchors_3d(&mut self)`（第 74–78 行）

清空当前场景的 draw call 锚点列表。

### `register_scene_draw_call_anchor_3d(&mut self, area: Area, world_pos: Vec3f)`（第 80–94 行）

注册一个新的 draw call 锚点：
1. 检查 `scene_3d_mut()` 是否存在，不存在则直接返回
2. 从 `Area` 中提取 `draw_list_id` 和 `draw_item_id`（仅 `Area::Instance` 包含这些信息，其他 `Area` 变体为 `None`）
3. 构造 `SceneDrawCallAnchor` 并推入 `scope.draw_call_anchors`

### `register_last_scene_draw_call_anchor_3d(...)`（第 96–111 行）

注册最后一个 draw call 的锚点，直接接受 `draw_list_id` 和 `draw_item_id` 而非 `Area`。`area` 字段设为 `Area::Empty`。

---

## 六、`Cx3dState`（第 28–29 行，定义在 `scene_3d.rs`）

```rust
pub(crate) struct Cx3dState {
    pub scene_scope: Option<SceneScope3D>,
}
```

这是一个内部辅助结构体，`pub(crate)` 可见性，将 `SceneScope3D` 包装为可选值。

---

## 七、使用模式总结

```
Cx3d 的典型生命周期:
1. new(cx)          — 创建 3D 上下文
2. begin_scene_3d() — 开始 3D 场景
3. set_scene_world_transform_3d() — 设置世界变换
4. register_scene_draw_call_anchor_3d() — 注册锚点（多次）
5. end_scene_3d()   — 结束 3D 场景
```

`Cx3d` 通过 `Deref` 透明继承 `CxDraw` 的所有 2D 绘图能力，使 3D 场景绘制代码可以无缝使用 2D 基础绘图 API。
