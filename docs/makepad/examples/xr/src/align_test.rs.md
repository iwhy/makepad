# `examples/xr/src/align_test.rs`

XR 空间对齐测试的可视化工具。用于验证 Makepad XR 的 `XrNetAlignmentDescriptorFrame` 算法——通过 TSDF 深度快照生成对齐描述符、计算本地与远程坐标系的变换矩阵，并将结果可视化。

## 核心功能

`AlignTest` 是一个 Makepad Widget，它：

1. 从 TSDF 深度数据生成对齐描述符（`XrNetAlignmentDescriptorFrame`）
2. 模拟"远程"描述符（通过施加已知 ground truth 变换）
3. 运行 `solve_remote_to_local` 解算对齐
4. 将解算结果（位置偏移、偏航角度）与实际 ground truth 比较
5. 在 3D 空间中绘制本地和远程标记（Markers），可视化对齐误差

## `script_mod!` 注册（行 9-19）

```rust
script_mod! {
    use mod.prelude.widgets_internal.*

    mod.widgets.AlignTestBase = #(AlignTest::register_widget(vm))
    mod.widgets.AlignTest = set_type_default() do mod.widgets.AlignTestBase{
        body: mod.widgets.XrBodyKind.Disabled
        draw_cube +: {
            light_dir: vec3(0.35, 0.8, 0.45)
        }
    }
}
```

- 使用 `widgets_internal` prelude（因为是 widget 开发）
- 基类型注册 Rust 结构体，`set_type_default()` 扩展脚本属性
- `body: Disabled` — 不参与物理模拟
- `draw_cube +: { light_dir }` — 设置自定义光照方向

## `AlignTest` 结构体（行 21-43）

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct AlignTest {
    #[redraw] #[live] draw_cube: DrawCube,
    #[rust] enabled: bool,
    #[rust] last_mesh_generation: u64,
    #[rust] last_mesh_update_sequence: u64,
    #[rust] last_status: String,
    #[rust] local_markers: Option<[Vec3f; 2]>,
    #[rust] remote_markers_local: Option<[Vec3f; 2]>,
    #[rust] last_solution: Option<XrNetAlignmentSolution>,
    #[cast] #[deref] node: XrNode,
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `draw_cube` | `DrawCube` | 用于在 3D 空间中绘制标记立方体 |
| `enabled` | `bool` | 是否启用了对齐测试 |
| `last_mesh_generation` | `u64` | 上一次处理的 TSDF 快照 generation 号 |
| `last_mesh_update_sequence` | `u64` | 上一次处理的 TSDF 更新序列号 |
| `last_status` | `String` | 当前状态文本 |
| `local_markers` | `Option<[Vec3f; 2]>` | 本地描述符的两个测试标记点（世界坐标） |
| `remote_markers_local` | `Option<[Vec3f; 2]>` | 远程描述符变换到本地坐标系后的标记点 |
| `last_solution` | `Option<XrNetAlignmentSolution>` | 最近一次解算结果 |
| `node` | `XrNode` | 继承自 XrNode 的行为能力 |

`#[cast]` 和 `#[deref]` 使 `AlignTest` 可以作为 XrNode 使用，`script_call` 和 `draw_3d` 的默认实现会委托给 `node`。

## 公共方法

### `status_text()` / `enabled()`（行 46-56）

简单的状态查询接口，供外部（如 UI 控件）读取测试状态。

### `set_enabled(cx, enabled)`（行 58-76）

启用或禁用对齐测试：

1. 如果状态未变化则返回
2. 调用 `cx.xr_tsdf().set_surface_analysis_enabled(enabled)` — 启用/禁用 TSDF 表面分析
3. 重置所有缓存数据（generation、sequence、solution、markers）
4. 设置状态文本

```rust
fn set_enabled(&mut self, cx: &mut Cx, enabled: bool) -> bool {
    // ...
    cx.xr_tsdf().set_surface_analysis_enabled(enabled);
    self.last_mesh_generation = 0;
    self.last_mesh_update_sequence = 0;
    self.last_solution = None;
    self.local_markers = None;
    self.remote_markers_local = None;
    // ...
}
```

### `refresh_alignment(cx, time)`（行 78-157）

对齐测试的核心算法：

```rust
fn refresh_alignment(&mut self, cx: &mut Cx, _time: f64) {
    if !self.enabled { return; }
    let Some(snapshot) = cx.xr_tsdf().latest_tsdf_snapshot() else { return; };
    // ...
}
```

**步骤分解**:

1. **跳过不变数据**（行 91-95）:
   ```rust
   let mesh_unchanged = snapshot.generation == self.last_mesh_generation
       && snapshot.update_sequence == self.last_mesh_update_sequence;
   if mesh_unchanged { return; }
   ```

2. **生成本地对齐描述符**（行 99-108）:
   ```rust
   let local_descriptor = XrNetAlignmentDescriptorFrame::from_tsdf_snapshot(snapshot.as_ref(), 0.0);
   ```

3. **定义 Ground Truth 变换**（行 109-116）:
   ```rust
   let ground_truth_translation = vec3f(-0.82, 0.0, 0.67);
   let ground_truth_yaw = 0.58f32;
   let ground_truth_remote_to_local = Pose::new(
       Quat::from_axis_angle(vec3f(0.0, 1.0, 0.0), ground_truth_yaw),
       ground_truth_translation,
   ).to_mat4();
   ```

4. **模拟远程描述符**（行 117）:
   对本地描述符施加 `local_to_remote` 逆变换，模拟"远程设备发送过来的描述符"：
   ```rust
   let remote_descriptor = local_descriptor.transformed(&local_to_remote);
   ```

5. **提取测试标记**（行 119-122）:
   分别提取本地和远程（变换回本地坐标）的标记点，用于可视化对比：
   ```rust
   self.local_markers = local_descriptor.test_markers();
   self.remote_markers_local = remote_descriptor.transformed(&ground_truth_remote_to_local).test_markers();
   ```

6. **解算对齐**（行 123-126）:
   ```rust
   self.last_solution = XrNetAlignmentDescriptorFrame::solve_remote_to_local(
       &local_descriptor,
       &remote_descriptor,
   );
   ```

7. **计算误差**（行 139-155）:
   - `position_error_cm`: 平移误差（厘米）
   - `yaw_error_deg`: 偏航角误差（度）
   - `overlap_error_cm`: 标记重叠误差
   - `solution.confidence, matched_samples`: 解算置信度和匹配样本数
   - `preview.local_sample_count, floor_sample_count, wall_sample_count`: 采样统计

## 3D 绘制

### `marker_color(index, alpha)`（行 166-172）

```rust
fn marker_color(index: usize, alpha: f32) -> Vec4f {
    match index {
        0 => vec4f(1.0, 0.20, 0.20, alpha),  // 标记 0: 红色
        1 => vec4f(0.22, 0.48, 1.0, alpha),  // 标记 1: 蓝色
        _ => vec4f(0.92, 0.92, 0.92, alpha), // 其他: 灰色
    }
}
```

本地标记为纯色（alpha=1.0），远程标记为半透明色（alpha=0.34），便于视觉区分。

### `draw_marker(cx, world, center, size, color)`（行 174-188）

在 3D 世界空间中绘制一个标记立方体：
```rust
fn draw_marker(&mut self, cx: &mut Cx3d, world: &Mat4f, center: Vec3f, size: Vec3f, color: Vec4f) {
    self.draw_cube.transform = Mat4f::mul(world, &Pose::new(Quat::default(), center).to_mat4());
    self.draw_cube.cube_pos = vec3f(0.0, 0.0, 0.0);
    self.draw_cube.cube_size = size;
    self.draw_cube.color = color;
    self.draw_cube.depth_clip = 1.0;
    self.draw_cube.draw(cx);
}
```

使用 `DrawCube` 绘制，通过 `transform` 矩阵将标记放置在世界空间中的正确位置。

## Widget 实现

### `script_call`（行 192-218）

公开三个脚本调用接口：

| 方法 ID | 功能 |
|---------|------|
| `set_enabled` | 设置启用状态（参数为 bool），返回新状态 |
| `toggle_enabled` / `toggle_test` | 切换启用状态，返回新状态 |
| `enabled` | 返回当前启用状态（getter） |

未匹配的方法委托给 `self.node.script_call`。

### `handle_event`（行 224-229）

```rust
fn handle_event(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope) {
    if let Event::XrUpdate(update) = event {
        self.refresh_alignment(cx, update.state.time);
    }
    self.node.handle_event(cx, event, scope);
}
```

在每帧 XR 更新事件中刷新对齐计算，其余事件委托给 XrNode。

### `draw_3d`（行 231-262）

```rust
fn draw_3d(&mut self, cx: &mut Cx3d, scope: &mut Scope) -> DrawStep {
    if !self.enabled { return self.node.draw_3d(cx, scope); }
    // ...
    // 绘制本地标记（实色）
    if let Some(local_markers) = self.local_markers { ... }
    // 绘制远程标记（半透明）
    if let Some(remote_markers_local) = self.remote_markers_local { ... }
    self.node.draw_3d(cx, scope)
}
```

1. 如果未启用，直接委托给 XrNode 绘制
2. 获取世界变换矩阵（`xr_widget_world_transform`）
3. 绘制两个本地标记（不透明，6cm 立方体）
4. 绘制两个远程标记（半透明，6.6cm 立方体，略大以示区分）
5. 继续绘制 XrNode 的子节点

### `draw_walk`（行 264-266）

```rust
fn draw_walk(&mut self, _cx: &mut Cx2d, _scope: &mut Scope, _walk: Walk) -> DrawStep {
    DrawStep::done()
}
```

2D 绘制直接跳过——AlignTest 只做 3D 渲染。

## 工具函数

### `marker_overlap_error`（行 269-274）

```rust
fn marker_overlap_error(local: Option<[Vec3f; 2]>, remote: Option<[Vec3f; 2]>) -> f32 {
    let (Some(local), Some(remote)) = (local, remote) else { return 0.0; };
    ((remote[0] - local[0]).length() + (remote[1] - local[1]).length()) * 50.0
}
```

计算本地和远程两个标记之间的平均距离（单位：厘米），乘以 50 转换为合适的度量。

### `wrap_angle`（行 276-284）

将角度归一化到 `(-PI, PI]` 范围。

## 总结

`AlignTest` 是一个**XR 空间对齐算法的验证工具**。它展示了：

1. **`XrNetAlignmentDescriptorFrame`** 的使用：从 TSDF 快照生成对齐描述符
2. **闭环验证**：用已知 ground truth 变换生成"远程"描述符，再解算对齐比较误差
3. **3D 可视化**：在场景中绘制标记点，直观显示对齐效果
4. **Widget 模式**：继承 XrNode、自定义绘制和事件处理
5. **XR 开发工作流**：对对齐算法的精度进行实时评估的工具化方式

主要用于开发和调试 XR 多设备空间对齐算法，通过 visual feedback 帮助开发者判断算法精度。
