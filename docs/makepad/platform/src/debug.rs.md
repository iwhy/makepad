# `debug.rs` — 调试可视化与绘制树诊断

## 用途

`debug.rs` 提供了两套调试设施：一套是基于 `Debug` 结构的**运行时可视化标记系统**，允许开发者在渲染过程中标记点、矩形、区域和文本标签以进行视觉调试；另一套是**绘制树递归打印工具** `debug_draw_tree`，用于分析渲染管线的 draw call 分层结构。

## Debug 结构体体系

### `DebugInner`
```rust
pub struct DebugInner {
    pub areas: Vec<(Area, Vec2d, Vec2d, Vec4f)>,
    pub rects: Vec<(Rect, Vec4f)>,
    pub points: Vec<(Vec2d, Vec4f)>,
    pub labels: Vec<(Vec2d, Vec4f, String)>,
    pub marker: u64,
}
```
存储调试几何数据的内部状态，通过 `Rc<RefCell<DebugInner>>` 实现共享可变状态。

### `Debug(Rc<RefCell<DebugInner>>)`
对外暴露的调试句柄，支持 `Clone`（所有克隆共享同一内部状态）。提供以下标记方法：

- **`point(p: Vec2d, color: Vec4f)`**: 在指定像素位置记录一个带颜色的点。
- **`label(p: Vec2d, color: Vec4f, label: String)`**: 在指定位置记录带颜色的文本标签。
- **`rect(p: Rect, color: Vec4f)`**: 记录一个矩形区域及其边框颜色。
- **`area(area: Area, color: Vec4f)`**: 记录一个 `Area` 的边界，偏移量默认 `(0,0)-(0,0)`。
- **`area_offset(area: Area, tl: Vec2d, br: Vec2d, color: Vec4f)`**: 记录带自定义偏移量的区域边界。

状态管理方法：
- **`marker() -> u64`**: 返回当前标记值。
- **`set_marker(v: u64)`**: 设置标记值，用于在调试数据流中标识版本。
- **`has_data() -> bool`**: 检查是否有任何未取出的调试数据。
- **`take_rects()` / `take_points()` / `take_labels()` / `take_areas()`**: 消耗性地取出各类调试数据（通过 `std::mem::swap` 交换出空 Vec），类型于通道的 `try_recv`，用于渲染循环中每帧消费一次。

## Cx::debug_draw_tree

### 函数签名
```rust
pub fn debug_draw_tree(&self, dump_instances: bool, draw_list_id: DrawListId)
```

### 实现逻辑

递归遍历从指定 `draw_list_id` 开始的绘制列表树，对每个节点输出：

1. **绘制列表节点**：输出 `debug_id`、`DrawListId`、和其中的 draw item 数量（`draw_item_order_len`）。缩进使用 `|   ` 风格表示树深度。

2. **Draw Item 叶子节点**：对于非子列表的 draw item，提取其 `DrawCall` 信息，输出：
   - `debug_id`（取自 `DrawCall` 的 `options.debug_id` 或着色器的 `debug_id`）
   - `sid`：着色器 ID（`draw_shader_id.index`）
   - `inst`：实例数量（`instances.len() / slots`）
   - `zbias`：Z 轴偏移量
   - `group`：draw call 分组 ID

3. **实例数据转储**（当 `dump_instances = true` 时）：
   - **动态 uniform 输入**：遍历着色器的 `dyn_uniforms.inputs`，根据每个输入的 `slots`（1/2/3/4）格式化为 `id:value`、`id:v2(x,y)`、`id:v3(x,y,z)` 或 `id:v4(x,y,z,w)`。
   - **实例数据**：对每个实例（最多输出第一个实例以保持输出简洁），遍历着色器的 `instances.inputs`，读取实例缓冲区中的值并用相同方式格式化。

### 辅助递归函数 `debug_draw_tree_recur`

使用纯字符串构建，通过 `depth` 参数控制缩进级别。递归深度通过 `sub_list()` 检查进入子绘制列表。输出通过 `log!` 宏打印。

## 设计要点

1. **非侵入式调试标记**：`Debug` 结构使用 `Rc<RefCell<>>` 实现共享可变状态，允许调试标记在任何位置插入到渲染管线中，而无需修改函数签名或引入全局变量。

2. **消费式读取模式**：`take_*` 方法使用 `swap` 取出数据，避免了锁竞争和重复处理问题。每帧渲染结束时，调试数据被消费一次，下一帧重新开始收集。

3. **绘制树诊断的缩进可视化**：递归缩进输出直观地展示了绘制列表的嵌套层次，配合 `debug_id` 可以快速定位渲染管线的分层问题和 draw call 分布。

4. **轻量级实例诊断**：通过限制实例 dump 到第一个实例，在提供足够调试信息的同时避免日志爆炸。uniform 和 instance 输入的多 format 处理（float/v2/v3/v4）使输出紧凑可读。

5. **逃逸方案**：被注释掉的 DrawList 空列表检查、标题和底部标记等代码是调试时快速启用的备选逻辑，展示了该功能的调试驱动开发背景。
