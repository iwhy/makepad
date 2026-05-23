# draw_list.rs — 绘制命令队列管理

## 概述

`draw_list.rs` 是 Makepad GPU 绘制管线的**命令队列管理层**。它定义了绘制列表 (`CxDrawList`)、绘制项 (`CxDrawItem`)、绘制调用 (`CxDrawCall`)、绘制列表 uniform 缓冲区 (`DrawListUniforms`) 以及绘制调用 uniform 缓冲区 (`DrawCallUniforms`) 等核心数据结构，并提供了 GPU 性能指标统计功能。

---

## 核心类型

### `DrawList` / `DrawListId`（第 23–48 行）

`DrawList` 是一个轻量级句柄，内部包裹一个 `PoolId`，通过 `IdPool` 从 `Cx` 中分配。`DrawListId` 包含 `(usize, u64)` 二元组作为世代编号，防止悬垂引用。

- **`DrawList::new(cx)`** — 通过 `cx.draw_lists.alloc()` 分配一个新的绘制列表池条目，返回句柄。
- **`DrawList::set_reset_zbias()`** — 设置当前绘制列表是否需要在绘制后重置 Z-bias。
- **`DrawList::id()`** — 从内部 PoolId 提取 `DrawListId`。
- **`DrawListId::index()` / `DrawListId::generation()`** — 分别获取池索引和世代号。

### `GpuPassMetrics`（第 50–59 行）

GPU 通道性能指标结构，包含：
- `draw_calls`: 绘制调用次数
- `instances`: 实例总数
- `vertices`: 顶点总数（索引数 × 实例数）
- `instance_bytes`: 实例数据上传字节
- `uniform_bytes`: uniform 数据上传字节
- `vertex_buffer_bytes`: 顶点缓冲区上传字节
- `texture_bytes`: 纹理上传字节

---

## Cx 方法：GPU 指标统计（第 61–293 行）

### `Cx::collect_gpu_pass_metrics()`（第 62–77 行）

入口函数，给定 `DrawPassId`，递归扫描该通道的主绘制列表，累计各类 GPU 资源上传量。使用 `HashSet<GeometryId>` 和 `Vec<TextureId>` 追踪已统计的几何体和纹理，避免重复计数。

### `Cx::estimate_texture_upload_bytes()`（第 79–195 行）

根据纹理格式估算单次上传的字节数：
- `VecBGRAu8_32`（BGRA 8-bit，32 对齐）：`width × height × 4`
- `VecCubeBGRAu8_32`（立方体贴图）：在上述基础上 × 6 个面
- `VecMipBGRAu8_32`（Mipmap BGRA）：同普通 BGRA，只计第一层
- `VecMipRGBAf32` / `VecRGBAf32`（RGBA float）：`width × height × 16`（4 × 4 字节）
- `VecRu8`（单通道 u8）：`width × height`
- `VecRGu8`（双通道 u8）：`width × height × 2`
- `VecRf32`（单通道 f32）：`width × height × 4`
- 跳过 `TextureUpdated::Empty` 的纹理（0 字节）

对各种纹理格式一一匹配，计算出合理的内存/带宽开销。

### `Cx::collect_gpu_metrics_for_draw_list()`（第 197–293 行）

递归遍历绘制列表的所有绘制项：
1. 若遇到 `SubList`，递归进入子列表。
2. 若遇到 `DrawCall` 且 `instance_slots > 0`，计算实例数 = `instances.len() / instance_slots`。
3. 获取几何体的索引数量并累加顶点数（`index_count × instance_count`）。
4. 若实例脏标记为 true，统计实例缓冲区字节（`instances.len() × 4`）。
5. 统计 uniform 字节：将 `DrawCallUniforms` + `PassUniforms` + `DrawListUniforms` + `dyn_uniforms` + `scope_uniforms_buf` 的 f32 数量累加，× 4 字节 × 2（VS+FS 各一份）。
6. 若几何体尚未上传，累加其顶点和索引缓冲区大小。
7. 遍历纹理槽，对尚未统计的纹理估算上传字节。

---

## `DrawCallUniforms`（第 354–399 行）

绘制调用级别的 uniform 数据，`#[repr(C)]` 布局确保与 GPU 端内存对齐：
- `zbias`: f32 — Z-bias 偏移
- `pad1`–`pad3`: 填充字段，对齐到 16 字节

**`as_slice()`** — 将结构体 reinterpret 为 `&[f32; N]`，用于上传到 GPU。

**`set_zbias()`** — 设置 Z-bias 值。

---

## `CxDrawKind` 枚举与 `CxDrawItem`（第 401–448 行）

绘制项的内部表示，有三种变体：
- `SubList(DrawListId)` — 子绘制列表引用
- `DrawCall(CxDrawCall)` — 实际的绘制调用
- `Empty` — 空占位，用于延迟填充

**辅助方法**：`is_empty()`、`sub_list()`、`draw_call()`、`draw_call_mut()`，方便安全地匹配枚举。

`CxDrawItem` 包含 `redraw_id`（用于脏判定）、`kind`（上述枚举）、`draw_item_id`（自增 ID）、`instances`（实例数据的 `Option<Vec<f32>>`）以及 `os`（平台相关的绘制调用数据）。

---

## `CxDrawCall`（第 451–481 行）

描绘一个 GPU 绘制调用的全部状态：
- `draw_shader_id`: 关联的着色器 ID
- `options`: 着色器选项（深度写入、Alpha 混合、背面剔除等）
- `append_group_id`: 合并分组的键值
- `total_instance_slots`: 每个实例占用的 f32 槽位数
- `draw_call_uniforms`: `DrawCallUniforms`（Z-bias 等）
- `geometry_id`: 几何体 ID
- `dyn_uniforms`: 256 个 f32 的用户自定义 uniform
- `texture_slots`: 16 个纹理槽
- `uniform_buffer_slots`: 2 个 uniform 缓冲区槽
- `instance_dirty` / `uniforms_dirty`: 脏标记，触发 GPU 上传

**`CxDrawCall::new()`** — 从 `DrawVars` 和着色器映射创建新的绘制调用，初始化所有字段，脏标记默认 true。

---

## `DrawListUniforms`（第 483–514 行）

绘制列表级别的 uniform，`#[repr(C)]` 布局：
- `view_transform`: Mat4f — 视图变换矩阵
- `view_clip`: Vec4f — 裁剪区域 `(min_x, min_y, max_x, max_y)`，默认极值 `(-100000, -100000, 100000, 100000)`
- `view_shift`: Vec2f — 视图偏移
- `pad1` / `pad2`: 填充

**`as_slice()`** — reinterpret 为 `&[f32; N]` 用于 GPU 上传。

---

## `CxDrawItems`（第 516–562 行）

用于管理 `CxDrawItem` 的缓冲池。使用 `used` 计数器跟踪实际使用量，避免频繁重新分配：
- **`push_item()`** — 如果池中有空闲槽则复用（保留 GPU 资源），否则分配新项。关键优化：复用时会清除实例数据但保留分配的内存。
- **`clear()`** — 将 `used` 置零，不释放内存。
- **`len()`** — 返回 `used` 计数。

---

## `CxDrawList`（第 564–585 行）

核心绘制列表结构：
- `debug_id` / `debug_dump` / `debug_dump_count`: 调试支持
- `reset_zbias`: 是否在列表结束时重置 Z-bias
- `codeflow_parent_id`: 用于代码流嵌套的父列表 ID
- `redraw_id`: 当前帧的重绘 ID
- `draw_pass_id`: 关联的通道
- `draw_items`: `CxDrawItems` — 绘制项集合
- `draw_item_reorder`: 可选的绘制项重排序
- `draw_list_uniforms`: 列表级 uniform
- `draw_list_has_clip`: 是否有裁剪
- `os`: 平台相关绘制列表数据
- `rect_areas`: 矩形区域列表，用于裁剪/命中测试
- `find_appendable_draw_shader_check`: 着色器 ID 哈希缓存，加速合并查找

---

## `CxDrawList` 核心方法（第 592–932 行）

### 分组追踪（第 610–648 行）

Ap 使用 `group_base()` 和 `group_lane()` 从 64 位 group ID 中提取基部和通道（低 8 位为 lane）。

**`can_cross_group_barrier()`** — 判断是否可以跨越分组屏障：
- 目标与屏障的分组基部不同时不能跨越
- 背景 lane（lane 0）可以跨越内容 lane（lane 1）的屏障
- 非默认 `draw_call_group` 的显式层可以跨越其他背景 lane 屏障以找到锚定调用

### `find_appendable_drawcall()`（第 650–809 行）

核心合并优化：从后向前遍历绘制项，查找可合并的现有绘制调用：
1. 若 `draw_shader_id` 为 None，直接返回。
2. 计算着色器对比键和分组 ID。
3. 逆向遍历绘制项：
   - 遇到 `SubList` 或 `Empty` 则停止搜索。
   - 若着色器索引不同且不能跨屏障，停止搜索。
   - 若着色器相同且 group 匹配：
     - 若 `draw_call_nocompare` 未设置，比较几何体、动态 uniform、纹理槽、uniform 缓冲区槽是否完全一致；若有差异且不能跨屏障则停止。
     - 比较选项是否可合并。
     - 全部匹配则返回该绘制项的索引。
   - 若存在屏障且不能跨越，停止搜索。

该算法的核心目的是将连续且状态相同的绘制调用合并为一个，减少 GPU 的 draw call 开销。

### `append_draw_call()`（第 811–837 行）

添加新的绘制调用：
1. 将着色器的 `false_compare_check()` 压入检查列表。
2. 调用 `push_item()` 分配一个 `CxDrawKind::DrawCall` 项。

### `draw_item_order_len()` / `draw_item_id_at_order_index()`（第 839–855 行）

处理可选的重排序：若存在 `draw_item_reorder`，则通过它间接访问绘制项；否则按原始顺序访问。

### `clear_draw_items()`（第 857–863 行）

重置绘制列表为初始状态：更新 `redraw_id`、清空子项、移除重排序、清空矩形区域和着色器检查列表。

### `append_sub_list()`（第 865–870 行）

添加一个子绘制列表引用作为绘制项。

### `store_sub_list_last()`（第 872–899 行）

将子列表存储到末尾位置：
1. 先扫描所有现有项，若发现自己已存在则将其置为 Empty。
2. 若最后一项是 Empty，复用该位置。
3. 若最后一项已是相同子列表，更新 `redraw_id`。
4. 否则 fallback 到 `append_sub_list()`。

这个复杂逻辑保证了子列表在绘制顺序中的正确位置。

### `store_sub_list()`（第 901–920 行）

较简单的子列表存储：先查重避免重复，再查找空的 Empty 槽复用，最后 fallback 到追加。

### `clear_sub_list()`（第 922–932 行）

将指定子列表 ID 对应的所有项标记为 Empty。
