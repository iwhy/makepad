# chart.rs — 图表组件（K 线 / 折线 / 柱状 / 散点）

## 整体职能
`Chart` widget 提供一套完整的金融/数据图表渲染方案，支持 **K 线图（Candlestick）**、**折线图（Line）**、**柱状图（Bar）** 和 **散点图（Scatter）** 四种图表类型。内部通过多种 Draw shader 实现高效的 GPU 绘制。

## 主要数据结构
- **`Chart`**：顶层 widget，包含 `draw_bg`、`draw_chart`、`draw_grid` 等绘制句柄，以及 `#[live]` 公开的数据字段如 `series`（数据序列）、`chart_type`（图表类型枚举）、`min` / `max`（Y 轴范围）等。
- **`ChartSeries`**：表示单条数据序列，包含 `label`、`data`（f32 数组）、`color` 及序列元数据。
- **`ChartType`**：枚举，取值 `Candlestick`、`Line`、`Bar`、`Scatter`。
- **`DrawChart`**：自定义 Draw shader，`#[repr(C)]` 布局，通过 `DrawVars` 批量上传实例数据到 GPU，支持高效的批量矩形/线段绘制。
- **`ChartGridLine`**：用于存储网格线的起点/终点/颜色信息。
- **`ChartTooltip`**：十字光标提示线的状态，包含 `cross_x` / `cross_y` 位置和 `draw_tooltip` 绘制句柄。

## 方法与实现逻辑

### `fn script_component` — 脚本注册
注册 `Chart` 及内部的 `DrawChart` 到 Makepad 脚本运行时。先注册 `DrawChart` 的 shader 类型（继承自 `DrawQuad`），再注册 `Chart` widget 本身。

### `fn draw_walk` — 核心绘制流程
1. 调用 `draw_bg.draw_abs(cx, rect)` 绘制背景。
2. 调用 `draw_grid_lines` 绘制网格。
3. 根据 `self.chart_type` 分发到不同的绘制方法：Candlestick 走 `draw_candlestick`，Line 走 `draw_line`，Bar 走 `draw_bar`，Scatter 走 `draw_scatter`。
4. 调用 `draw_tooltip` 绘制十字光标和数值提示。
所有绘制基于 `DrawChart` 的统一批量上传机制：计算每个数据点的屏幕坐标和颜色，填充到 `DrawVars` 实例缓冲，最后通过 `draw_abs` 一次性提交到 GPU。

### `fn handle_event` — 事件处理
处理鼠标事件：鼠标移动时更新 tooltip 的 `cross_x` / `cross_y` 坐标；鼠标离开图表区域时隐藏 tooltip。通过 `hit_test` 判断鼠标是否在图表区域内。

### `fn draw_candlestick` — K 线图绘制
遍历 `ChartSeries` 的 OHLC 数据点，对每根 K 线计算实体（开盘-收盘矩形）和影线（最高-最低线段）的屏幕坐标。上涨阳线用 `color_up`、下跌阴线用 `color_down` 分别着色。通过 `DrawChart` 的实例缓冲一次性提交所有矩形和线段。

### `fn draw_line` — 折线图绘制
将数据序列中的每个点映射到屏幕坐标，生成连续的线段列表。支持多条序列叠加，每条序列使用各自的 color。最终以 `draw_line_list` 方式提交 GPU。

### `fn draw_bar` — 柱状图绘制
将每个数据点绘制为从 X 轴基线延伸的垂直矩形。正值向上、负值向下。柱状宽度根据数据点数量自动计算间距。支持多序列分组并排显示。

### `fn draw_scatter` — 散点图绘制
每个数据点绘制为一个小圆点（通过 DrawChart 的小矩形模拟）。支持调节点的大小。点的颜色取自序列的 color 或 Chart 的 `color_dot` 属性。

### `fn draw_grid_lines` — 网格线绘制
根据 Y 轴的范围（`min` / `max`）和刻度数量，计算水平网格线的位置并绘制。网格线的颜色和宽度由 `draw_grid` 的 shader 属性控制。

### `fn draw_tooltip` — 十字光标绘制
根据鼠标当前 `cross_x` 位置，找到最近的数据点索引，显示该点的数值。绘制一条垂直的十字线。提示标签跟随鼠标横向移动。

### `fn map_to_screen` — 数据坐标到屏幕坐标转换
将数据空间的 `(x, y)` 值映射到像素空间的 `(px, py)`。X 轴根据数据点数量均匀分布，Y 轴根据 `min` / `max` 线性映射。支持 `#[rust]` 级别的高效计算。

### `fn hit_test` — 命中检测
检查给定的屏幕坐标是否位于图表的绘制区域内，用于判断是否应该显示 tooltip。
