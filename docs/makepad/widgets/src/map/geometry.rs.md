# map/geometry.rs — 地图几何数据处理

## 整体职能
该文件提供地图渲染所需的底层几何计算工具集，包括 **线段简化**、**多边形三角剖分**、**向量瓦片要素解码** 以及 **屏幕空间几何绘制**。所有函数均为纯计算，不耦合 UI 状态。

## 核心函数与实现逻辑

### `fn decode_tile_geometry` — 解码向量瓦片几何体
接收 Protocol Buffer 格式的几何编码数据（来自 Mapbox Vector Tile 规范）。解析 `MoveTo` / `LineTo` / `ClosePath` 命令序列，还原为屏幕空间坐标。处理参数坐标（`parameter_size`）与实际像素的比例换算。返回 `DecodedGeometry` 枚举，区分 `Polygon`（多边形）、`LineString`（折线）和 `MultiPoint`（多点）。

### `fn simplify_geometry` — Ramer-Douglas-Peucker 线段简化
使用 RDP 算法对折线进行简化，减少顶点数量同时保持视觉形状。接收 `epsilon` 容差参数（像素单位）：容差越大，简化程度越高。递归实现：从首尾点构成的基线开始，找到距离基线最远的点，如果距离大于 epsilon 则保留并递归细分两侧，否则丢弃中间所有点。

### `fn triangulate_polygon` — 耳切三角剖分
对简单多边形执行 ear-clipping 三角剖分，将多边形分解为三角形列表，供 Draw shader 进行 GPU 绘制。选择凸顶点（内角 < 180°）作为 "ear"，检查 ear 内是否包含其他顶点，确认后切除并记录三角形。处理自相交多边形的退化情况。返回 `Vec<Vec2d>`（每三个 vec2d 为一个三角形）。

### `fn build_screen_polyline_into` — 构建屏幕坐标折线
将瓦片本地坐标的路径点（`&[i32]` 编码）转换为屏幕空间的 `Vec<Vec2d>`。应用 `scale` 缩放因子和 `map_offset` 平移。复用传入的可变引用以避免分配。

### `fn polyline_cumulative_lengths_into` — 折线累计长度
计算折线各分段在屏幕空间中的累计长度，结果写入 `scratch_cumulative`。每个元素代表从路径起点到该顶点的总长度。用于标注定位和碰撞检测。

### `fn sample_polyline_point_at_distance` — 按距离采样路径点
给定折线的累计长度数组，在指定的距离 `d` 处线性插值采样一个路径点。处理折线完全闭合的情况。返回 `Option<Vec2d>`。

### `fn sample_polyline_tangent_angle_raw` — 采样路径切线角
在距离 `d` 处，取前后各 `probe_delta` 距离的两个点，计算它们之间的方向角（弧度）。用于标签文字沿路径的旋转对齐。

### `fn polyline_outside_rect` — 折线是否在矩形外
检查折线的所有顶点是否都在给定的矩形之外，如果是则不需要绘制/标注。用于视锥剔除 (frustum culling)。

### `fn polyline_point_outside_rect` — 点是否在矩形外
快速检查单个点是否在矩形之外，带有 `margin` 边距。

### `fn polyline_bounds_from_points` — 计算点集边界框
遍历点集，计算最小和最大 X/Y 值，返回 `Rect`。

### `fn smooth_label_curve_into` — 标注曲线平滑
对候选标注路径进行 Chaikin 曲线细分或迭代平滑，减少折线的锐角转折，使沿路径排列的文字更加平滑自然。使用两个 scratch 缓冲进行交替迭代。

### `fn choose_label_start_distance` — 选择标注起始距离
在平滑后的路径上选择一个合适的起始位置放置标签文字，使得文字居中于路径且不超出端点。如果路径太短放不下文字则返回 `None`。

### `fn choose_label_reverse` — 判断是否需要翻转文字
根据路径在标注位置附近的切线角度，确定文字是否需要旋转 180° 以避免文字倒置。当切线方向指向右半平面时不翻转，指向左半平面时翻转。

### `fn repeat_distance_for_label` — 计算标签重复距离
根据标签的优先级和源数据层级（source_rank），计算同一名称的标签在路径上可以重复出现的间距。优先级越高、层级越重要，重复间距越小（允许出现更频繁）。

### `fn label_source_rank` — 标签源数据层级权重
将 `source_layer`（OSM 中的 `water`、`road`、`poi` 等）映射为数值权重，用于标签评分排序。权重越高，在碰撞竞争中越优先显示。

### `fn normalize_label_key` — 标签键规范化
将标签文本转为小写并去除首尾空白，生成用于去重和碰撞检测的规范键。

### `fn rects_overlap_with_padding` — 带边距的矩形重叠检测
判断两个矩形是否重叠，考虑额外的 `padding` 内边距。用于标签碰撞检测。

### `fn clip_polygon_to_rect` — 多边形矩形裁剪
使用 Cohen-Sutherland 或 Sutherland-Hodgman 算法将多边形裁剪到视口矩形内，减少不必要的绘制。

### `fn extend_rect_to_include_point` — 扩展矩形包含点
如果点在当前矩形外，扩展矩形边界以将其包含在内。

## 绘制 Shader 注册

### `fn register_types` — 注册绘制类型
注册 `DrawMapPolygon`、`DrawMapLine`、`DrawMapPoint`、`DrawMapFill` 等自定义 Draw shader 到脚本运行时。这些 shader 继承自基础 Draw shader，添加了地图特有的 uniform（如 `u_map_offset`、`u_zoom`）。
