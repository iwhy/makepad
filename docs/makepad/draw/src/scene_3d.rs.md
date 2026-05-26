# `scene_3d.rs` — 3D 场景节点管理

**文件路径**: `draw/src/scene_3d.rs` (43 行)  
**核心作用**: 定义 3D 场景渲染所需的数据结构，包括场景状态、场景作用域、DrawCall 锚点和上下文状态。这些类型被 `cx_3d.rs` 使用，构成 3D 渲染管线的数据层。

---

## 一、`SceneState3D`（第 6–13 行）

```rust
#[derive(Clone, Copy, Debug, Default)]
pub struct SceneState3D {
    pub time: f64,              // 场景时间（秒），用于动画
    pub camera_pos: Vec3f,      // 相机世界位置
    pub view: Mat4f,            // 视图矩阵（View Matrix）
    pub projection: Mat4f,      // 投影矩阵（Projection Matrix）
    pub viewport_rect: Rect,    // 视口矩形
}
```

**字段说明**:
- `time`: 场景时间戳（`f64`），供 shader 中的动画使用（如顶点动画、纹理动画）
- `camera_pos`: 相机在世界空间中的位置，用于光照计算和 LOD 选择
- `view`: 视图矩阵，将世界坐标转换到观察空间（camera space）
- `projection`: 投影矩阵，将观察空间转换到裁剪空间（clip space），可以是透视或正交投影
- `viewport_rect`: 屏幕视口矩形，定义渲染区域的位置和尺寸

**实现特质**: `Clone`, `Copy`, `Debug`, `Default` — 轻量值类型，可自由复制。

---

## 二、`SceneDrawCallAnchor`（第 15–21 行）

```rust
#[derive(Clone, Copy, Debug)]
pub struct SceneDrawCallAnchor {
    pub area: Area,                      // 渲染区域
    pub draw_list_id: Option<DrawListId>,// 绘制列表 ID
    pub draw_item_id: Option<usize>,     // 绘制项 ID
    pub world_pos: Vec3f,                // 3D 世界坐标
}
```

**作用**: 将 3D 空间中的某个位置与 2D UI 绘制调用的区域关联起来。当 UI 需要知道某个 3D 对象在屏幕上的对应位置时（如标签、工具提示、标记点），使用此结构体建立映射。

**字段说明**:
- `area`: 该锚点在 2D 屏幕上的区域（`Area::Instance` 或 `Area::Rect`）
- `draw_list_id`: 关联的绘制列表 ID（某些场景下可能未知，用 `Option` 表达）
- `draw_item_id`: 关联的绘制项在列表中的索引
- `world_pos`: 对应的 3D 世界坐标

---

## 三、`SceneScope3D`（第 23–38 行）

```rust
#[derive(Clone, Debug, Default)]
pub struct SceneScope3D {
    pub scene: SceneState3D,                 // 场景状态
    pub world_transform: Mat4f,              // 当前世界变换矩阵
    pub draw_call_anchors: Vec<SceneDrawCallAnchor>,  // DrawCall 锚点列表
}
```

**作用**: 表示一个活跃的 3D 场景作用域。当 `Cx3d` 开始一个 3D 场景时，创建一个 `SceneScope3D` 存储当前快照。

**字段说明**:
- `scene`: 场景状态的不可变快照（视图矩阵、投影矩阵等在整个场景绘制期间保持不变）
- `world_transform`: 可变的当前世界变换矩阵。场景中可能存在多个对象，每个对象可能有不同的世界变换。此字段在 `Cx3d::set_scene_world_transform_3d()` 中更新
- `draw_call_anchors`: 当前场景已注册的所有 DrawCall 锚点列表。随着场景对象的绘制不断追加

**`SceneScope3D::new(scene: SceneState3D)`**（第 31–37 行）：
构造函数，初始化 `scene`、`world_transform` 为单位矩阵、`draw_call_anchors` 为空向量。

---

## 四、`Cx3dState`（第 40–43 行）

```rust
#[derive(Default)]
pub(crate) struct Cx3dState {
    pub scene_scope: Option<SceneScope3D>,
}
```

**作用**: `pub(crate)` 可见性的内部状态包装器。存在于 `Cx3d` 结构体中，通过 `Option` 表达"当前是否处于 3D 场景绘制中"。

当 `scene_scope` 为 `Some` 时，表示 3D 场景已开始且未结束；为 `None` 时表示不在 3D 场景中。

---

## 五、数据流与生命周期

```
begin_scene_3d()
  |
  v
SceneScope3D::new(scene)   -- 创建作用域，固定 scene 状态
  |
  +-- set_world_transform()  -- 更新世界变换
  +-- register_anchor()      -- 添加锚点（多次）
  |
  v
end_scene_3d()             -- 丢弃作用域
  |
  v
Cx3dState::default()       -- 回到空状态
```

---

## 六、与 `cx_3d.rs` 的协作

`scene_3d.rs` 定义数据，`cx_3d.rs` 定义操作。对应关系：

| `scene_3d.rs` 数据类型 | `cx_3d.rs` 操作方法 |
|------------------------|-------------------|
| `SceneState3D` | `begin_scene_3d()` 的输入 |
| `SceneScope3D` | 内部存储，通过 `scene_3d()`/`scene_3d_mut()` 访问 |
| `Cx3dState` | `Cx3d` 的字段，生命周期管理 |
| `SceneDrawCallAnchor` | `register_scene_draw_call_anchor_3d()` 创建 |
