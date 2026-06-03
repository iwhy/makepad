# `desktop_profiler_view.rs` — Studio 性能分析视图

## 文件作用

实现图形化的性能分析面板，显示构建运行时的 GPU 帧时间、GPU 计数、GPU 上传字节和事件/GC 采样的可视化图表。支持实时跟随和时间轴拖拽/缩放。

## script_mod 定义

### `DesktopProfilerEventChart`

注册为独立 widget，包含多个绘制层：
- `draw_bg` — 背景
- `draw_line` — 时间网格线
- `draw_item` — 事件块和图表元素
- `draw_vector` — GPU 折线图
- `draw_label` / `draw_time` — 标签和时标文本

### `DesktopProfilerView`

包含完整的分析面板 UI：
```
View
 ├── Toolbar
 │   ├── running_button (ToggleFlat) — 运行/暂停开关
 │   ├── clear_button (ButtonFlat) — 清除数据
 │   ├── Filler
 │   └── stats (View)
 │       ├── status_label (P) — "Build: {name} ({id})"
 │       ├── sample_count_label (P) — "App E: {n} G: {n} C: {n}"
 │       └── window_label (Pbold) — "Live" / "Paused"
 │
 └── chart_scroll (ScrollYView)
     └── chart (DesktopProfilerEventChart)
```

## 常量

- `DEFAULT_PROFILE_WINDOW_SECONDS = 0.5` — 默认时间窗口
- `MIN_PROFILE_WINDOW_SECONDS = 0.00001` — 最小时间窗口
- `PROFILE_ROW_Y_STEP = 25.0` — 事件显示行间距
- `PROFILE_GRAPH_LANE_HEIGHT = 56.0` / `LANE_GAP = 10.0` — 折线图行高和间距
- `FRAME_BUDGET_SECONDS = 1/60` / `FRAME_BUDGET_120HZ_SECONDS = 1/120` — 帧预算线

## `DesktopProfilerEventChart`

### 结构体

```rust
struct DesktopProfilerEventChart {
    #[rust] time_range: TimeRange,      // 当前显示时间窗口
    #[rust] time_drag: Option<TimeRange>, // 拖拽起始时间窗口
    #[rust] follow_live: bool,           // 是否跟随最新数据
    // 绘制层: draw_bg, draw_line, draw_item, draw_vector, draw_label, draw_time
}
```

### 方法

**`profiler_build_id_from_context`**: 从 widget tree 路径反向查找 build_id。

**`set_follow_live`**: 切换实时跟随模式，清除拖拽状态。

**`sync_live_window`**: 根据最新采样时间调整时间窗口。

**`format_time_to_now_label`**: 格式化时间标签（`now` / `-500ms` / `-1.50s`）。

**`draw_time_grid`**: 绘制时间网格线：
- 自适应主刻度间距（保证 90-180px 间距）
- 主刻度线（2px 宽）+ 时标标签
- 次刻度线（1px 宽）

**`graph_plot_range_with_edges`**: 计算折线图的可见采样范围，包含边界外各一点用于连接线。

### 事件/采样显示

**`draw_profile_store`**: 显示三层事件数据：
1. **事件采样**（彩色块）：每个 `EventSample` 使用 `event_u32` 的哈希作为颜色
2. **GPU 采样**（灰色块）：显示 draw calls / instances / vertices 等统计
3. **GC 采样**（绿色块）：显示 heap_live 元数据

**`draw_block`**: 绘制单个采样块 + 标签（显示名称/持续时间/元数据）。

### 折线图

**`draw_graph_lane_background`**: 绘制图表轨道背景 + 水平网格线。

**`draw_gpu_frametime_graph`**: GPU 帧时间折线图：
- Y 轴：帧时间（ms），自动缩放
- 120Hz 预算线（红色标注）
- 橙色折线

**`draw_gpu_counts_graph`**: GPU 计数折线图：
- 三条线：Draw Calls（橙色）、Instances（蓝色）、Vertices（绿色）
- 右侧图例：D / I / VxI

**`draw_gpu_upload_graph`**: GPU 上传字节折线图：
- 四条线：Instance（橙色）、Uniform（蓝色）、Vertex Buffer（绿色）、Texture（粉色）
- 右侧图例：I / U / V / T

### Widget trait

**`draw_walk`**: 完整绘制流程：
1. 从 scope data 获取 build_id 和采样数据
2. 如果实时跟随，更新时间窗口
3. 绘制时间网格
4. 绘制事件块
5. 绘制三层折线图

**`handle_event`**: 时间轴交互：
- `FingerDown`（非跟随模式时）→ 开始拖拽时间轴
- `FingerMove` → 时间轴平移
- `FingerScroll` → 时间轴缩放（以鼠标位置为中心或右端）

## `DesktopProfilerView`

### 结构体

```rust
pub struct DesktopProfilerView {
    #[deref] view: View,
    tmp_status_label: String,         // 状态标签缓冲区
    tmp_sample_count_label: String,   // 采样计数标签缓冲区
}
```

### WidgetMatchEvent

**`handle_actions`**:
- `clear_button` 点击 → `DesktopProfilerViewAction::Clear`
- `running_button` toggle → 设置 `follow_live` + `DesktopProfilerViewAction::SetRunning`

### Widget trait

**`draw_walk`**:
1. 从 scope data 获取 build_id 和 profiler 状态
2. 更新 running_button 状态
3. 同步 chart 的 follow_live
4. 格式化状态标签文本
5. 委托 `view.draw_walk_all` 绘制

**`handle_event`**:
委托 `view.handle_event` + `widget_match_event`。

### Action 枚举

```rust
pub enum DesktopProfilerViewAction {
    SetRunning { build_id: QueryId, running: bool },
    Clear { build_id: QueryId },
    None,
}
```

## 与 Hub 的交互

- Profiler 数据通过 `HubToClient::QueryProfilerResults` 持续流入
- `App` 层通过 `start_profiler_query_for_build` / `stop_profiler_query_for_build` 控制采样生命周期
- 启动/暂停操作通过 `DesktopProfilerViewAction` 由 `App::handle_profiler_actions` 处理
