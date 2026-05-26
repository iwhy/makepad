# map/label.rs — 地图路径标注（碰撞检测与布局）

## 整体职能
该文件实现地图道路名称的 **沿路径文本标注** 系统，包括字体字形准备、路径碰撞检测 (QuadTree)、标签去重、以及字形沿曲线排列的 GPU 绘制。

## 核心类型与常量

- **`LabelCandidate`**：候选标签，包含文本、规范化键名(`name_key`)、道路种类(`road_kind`)、源数据权重(`source_rank`)、评分(`score`)、路径长度、中心点、重复间距(`repeat_distance`)、字体缩放因子(`font_scale`)和屏幕路径点集(`screen_path`)。
- **`LabelPerfStats`**：性能计数器，记录每一帧标签处理过程中各阶段的数量，包括扫描数、候选数(总量/保留)、Shaping 数(成功/尝试/预算)、绘制数(标签/字形)、以及各类拒绝原因(重复、过短、无可放置方案、超出视口、碰撞、预算超限)。
- **`DrawPathLabel`**：自定义 Draw shader (`#[repr(C)]`)，继承 `DrawVector`（矢量路径绘制），添加实例属性 `font_size`、`font_scale`、`color`、`halo_color`、`halo_radius`。负责将字形沿路径渲染到屏幕。
- 常量：`LABEL_COLLISION_PADDING`(12px 碰撞内边距)、`LABEL_VIEW_MARGIN`(50px 视口边距)、`LABEL_MIN_PATH_PIXELS`(60px 最短路径)、`LABEL_BASELINE_SHIFT_FACTOR`(0.30 基线偏移因子)、`LABEL_MAX_GLYPH_TURN_RADIANS`(0.65 字形最大转角)、`LABEL_GLYPH_ANGLE_BLEND`(0.25 角度融合系数)、`LABEL_MAX_CANDIDATE_COST`(1e9 最大候选代价)。

## 方法与实现逻辑

### `fn register_types` — 类型注册
将 `DrawPathLabel` 注册为脚本可用的 Draw shader 类型，路径标注相关的枚举和结构体也注册到脚本运行时。

### `fn draw_label` — 标签主绘制方法
1. 清零 `scratch_candidates`、`scratch_accepted_plans`、`path_glyphs` 等工作缓冲。
2. 调用 `collect_label_candidates` 遍历所有可见瓦片，筛选出可行的标签候选。
3. 将候选按评分降序排列，分数高的标签优先显示。
4. 遍历排序后的候选，对每个调用 `build_label_placement` 尝试在路径上放置文字：
   - **重复检测**：检查同一 `name_key` 是否已在该距离内出现（`repeat_distance` 控制），是则拒绝并记 `rejected_repeat`。
   - **预检查**：路径短于 `LABEL_MIN_PATH_PIXELS` 或无放置方案时提前拒绝。
   - **视口裁剪**：文字中心点超出 `rect + LABEL_VIEW_MARGIN` 时拒绝。
   - **碰撞检测**：与 `scratch_accepted_bounds` 中已接受的标签做带 `LABEL_COLLISION_PADDING` 的重叠检测。
   - **接受**：记录中心点（用于重复检测）、边界框（用于碰撞检测）和字形范围。
5. 根据评分排序后，调用 `draw_path_glyphs` 批量提交 GPU 字形绘制。
6. 更新 `label_perf` 计数。

### `fn collect_label_candidates` — 收集候选标签
遍历 `draw_tiles` 中的每个瓦片：
1. 跳过未就绪或无标签的瓦片。
2. 计算缩放因子 `scale = 2.0^(view_zoom - tile_zoom)` 和缩放差异 `zoom_delta`。
3. 对每个标签，检查 `source_rank` 有效性，规范化 `name_key`。
4. 将路径点转为屏幕坐标，检查视口裁剪。
5. 计算路径长度，短于阈值则跳过。
6. 在路径中点采样一个屏幕点，检查是否在视口内。
7. 根据优先级和 source_rank 计算 `repeat_distance` 和 `font_scale`。
8. 计算综合评分 `score`（考虑 source_rank、优先级、zoom_delta、路径长度）。
9. 复用 `scratch_candidates` 缓冲的已有分配，避免重新分配内存。

### `fn build_label_placement` — 构建标签放置方案
1. 对候选路径执行平滑处理（调用 `smooth_label_curve_into`），减少锯齿。
2. 准备字体字形 run（`prepare_single_line_run`）。
3. 计算平滑路径的累计长度，选择标签起始距离（确保文字在路径范围内居中）。
4. 采样路径中点处的切线方向，判断是否需要翻转文字（防止倒置）。
5. 计算基线偏移，使文字垂直居中于路径。
6. 调用 `place_text_along_path` 沿路径放置字形，生成 `PathTextPlacement`。

### `fn choose_label_start_distance` — 选择起始距离
从路径中点到两端搜索，找到可以容纳整个文字宽度的区间。如果路径总长度不足文字宽度，返回 `None`。

### `fn choose_label_reverse` — 判断翻转
当切线方向指向左半平面（角度在 π/2 到 3π/2 之间）时返回 `true`，将文字旋转 180° 使其始终朝向阅读方向。

### `fn repeat_distance_for_label` — 重复间距
优先级 1（最高）返回 `repeat_distance * 0.8`，优先级 2 返回 `repeat_distance`，其他返回 `repeat_distance * 1.4`。

### `fn label_source_rank` — 源层级权重
根据 `source_layer` 字符串返回权重值：`water`(1)、`road`(2)、`poi`(3)、`building`(4)、`landuse`(5)、`boundary`(6)、`place`(7)、`transit`(8)、`other`(9)。数值越小优先级越高。

### `fn normalize_label_key` — 键规范化
转小写，去除首尾空白。如果结果长度 < 2 则返回空字符串。
