# map/view.rs — 地图视图组件

## 整体职能
`MapView` 是地图模块的 **顶层 Widget**，负责整合瓦片管理、视口变换（平移/缩放/旋转）、地理要素绘制和道路标签渲染。它提供一个完整的矢量地图交互控件，支持鼠标/触摸/键盘操作和双主题（深色/浅色）。

## 主要数据结构
- **`MapView`**：顶层 widget，持有 `draw_bg`（背景）、`draw_fill`（面要素绘制）、`draw_line`（线要素绘制）、`draw_label`（标签绘制）、`touch_gesture`（触摸手势）、`animator`（动画控制）、`tiles`（`HashMap<TileKey, TileEntry>` 缓存）、`status`（状态文本）等。
- **`MapDragAnim`**：`Animator` 派生的动画控制器，管理 `drag` 和 `release` 动画状态，用于地图拖拽后的惯性滑动。
- **`MapZoomAnim`**：管理缩放动画，支持双指捏合和双击缩放。
- 视图状态参数：`center`（视口中心经纬度）、`zoom`（缩放级别）、`rotation`（旋转角）、`map_offset`（屏幕偏移量）。

### 工作缓冲
`MapView` 维护多个 `scratch_*` 缓冲（如 `scratch_candidates`、`scratch_accepted_bounds`、`scratch_accepted_centers`、`scratch_smooth_a/b`、`scratch_cumulative` 等），在帧间复用内存分配以减少 GC 压力。

## 方法与实现逻辑

### `fn register_widget` — Widget 注册
将 `MapView` 注册到脚本运行时，同时注册 `DrawMapFill`、`DrawMapLine`、`DrawMapPoint` 等绘制 shader。定义默认样式：地图背景为浅灰，填充色半透明蓝，线条为灰色。

### `fn draw_walk` — 核心绘制流程
1. 更新瓦片可见列表：调用 `get_visible_tiles` 计算当前视口需要显示的瓦片。
2. 请求缺失的瓦片：遍历 `visible_tiles`，对不在 `tiles` 缓存中的发送 `request_tile`。
3. 绘制背景 `draw_bg`。
4. 对每个可见瓦片中状态为 `Ready` 的条目，按 `DrawGroup` 顺序绘制：
   - **Fill pass**：调用 `draw_fill` 绘制多边形填充（水系、绿地、建筑等）。
   - **Line pass**：调用 `draw_line` 绘制线要素（道路、行政边界等）。
   - **Point pass**：调用 `draw_point` 绘制点要素（POI 图标等）。
5. 调用 `draw_label` 进行标签碰撞检测和路径文字绘制。
6. 调用 `update_status_text` 更新叠加的状态文本（调试信息）。
7. 调用 `draw_status.draw_abs(cx, status_rect)` 绘制状态文本覆盖层。

### `fn handle_event` — 事件处理
分发到多个子处理器：
- **鼠标/触摸**：平移地图（`handle_pan`）、缩放（`handle_zoom` — 滚轮或双指捏合）。
- **键盘**：方向键微移地图（`handle_keyboard`）、+/- 缩放。
- **动画帧**：驱动 `MapDragAnim` 和 `MapZoomAnim` 平滑过渡。
- **手势 Action**：处理 `TouchGesture` 发出的拖拽和惯性滑动 Action。

### `fn handle_pan` — 平移处理
将鼠标/触摸的像素移动量转换为地图坐标偏移：
1. 计算当前缩放级别下每像素对应的地理距离。
2. 更新 `center` 经纬度，将像素位移 `(dx, dy)` 转为经纬度增量。
3. 应用边界限制（最小/最大经纬度范围）。
4. 更新 `map_offset` 屏幕偏移量。
5. 如果正在惯性滑动（`MapDragAnim`），每帧调用 `handle_pan` 递减速度。

### `fn handle_zoom` — 缩放处理
1. 滚轮缩放：根据 `scroll_delta` 计算目标缩放级别，以鼠标位置为缩放中心。
2. 双指捏合：根据两指距离变化计算缩放因子。
3. 双击缩放：缩放到点击位置，放大一级。
4. 更新 `zoom` 值，触发 `MapZoomAnim` 动画。
5. 以缩放中心点不变为原则调整 `center` 和 `map_offset`。

### `fn set_center_zoom` — 设置中心与缩放
程序化设置视口中心和缩放级别，支持带动画过渡。当 `animate` 参数为 true 时启动 `MapZoomAnim`。

### `fn draw_tile_features` — 绘制瓦片要素
对单个瓦片中的 `TileFeature` 列表进行批量绘制提交：
1. 根据要素的 `source_layer` 从 `DrawStyle` 列表中查找匹配样式。
2. 如果是多边形要素，解码几何后进行耳切三角剖分，提交到 `draw_fill` 的实例缓冲。
3. 如果是线要素，解码几何后直接提交到 `draw_line` 的实例缓冲。
4. 批量提交时计算屏幕坐标（应用 `scale` 和 `map_offset`）。
5. 如果要素被标记为标签（道路名称），在 `labels` 列表中记录。

### `fn draw_label` — 标签绘制
1. 调用 `collect_label_candidates` 从可见瓦片收集所有候选标签。
2. 按评分降序排序，高评分（高优先级、长路径）标签优先。
3. 对每个候选调用 `build_label_placement` 沿路径放置文字：
   - 重复检测：同一 `name_key` 在 `repeat_distance` 内只保留一次。
   - 边界裁剪：标签位置超出视口边距则跳过。
   - 碰撞检测：与已放置标签的边界框做重叠检查。
4. 将接受的标签按评分排序后提交到 `draw_label.draw_path_glyphs`。
5. 详细性能计数写入 `label_perf`。

### `fn collect_label_candidates` — 构建候选列表
遍历 `draw_tiles` 中所有 `Ready` 状态瓦片：对每个瓦片的标签，计算缩放、屏幕路径、长度、评分等字段，填充 `scratch_candidates`。复用缓冲以避免分配。

### `fn build_label_placement` — 生成标签放置
对单个候选路径做平滑处理、准备字形、计算切线方向和基线偏移，调用 `place_text_along_path` 沿路径精确排列字形。返回 `PathTextPlacement`。

### `fn update_status_text` — 调试状态文本
收集各阶段计数（就绪/加载中/失败瓦片数、标签各阶段数量），格式化为一行调试文本。如果计数相比上一帧无变化，跳过 `format!` 以减少分配。

### `fn view_zoom` — 限制缩放范围
将 `self.zoom` 限制在 `[min_zoom, max_zoom]` 范围内，默认 `min_zoom=0`，`max_zoom=18`。本地模式时限制为 `[LOCAL_MBTILES_MIN_ZOOM, LOCAL_MBTILES_MAX_ZOOM]`。

### `fn request_zoom_level` — 请求的瓦片缩放级别
取 `view_zoom` 的四舍五入整数值。本地模式下限制为 `[0, 14]`（MBTiles 典型上限）。

### `fn source_mode_label / theme_label` — 状态标签
`source_mode_label` 返回 `"offline"` / `"online"` / `"disabled"`；`theme_label` 返回 `"dark"` / `"light"`。

## 注册的 Draw Shader
- **`DrawMapFill`**：渲染多边形要素（水面、绿地、建筑填充），支持 `color` 和 `opacity` uniform。
- **`DrawMapLine`**：渲染线要素（道路、边界、等深线），支持 `color`、`width`、`dash_pattern` uniform。
- **`DrawMapPoint`**：渲染点要素（POI 图标），支持 `sprite` 纹理和 `size` uniform。
