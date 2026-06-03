# charts/src/main.rs

演示 Makepad 的四种图表组件。

## 整体结构

- **第7-49行**：`script_mod!` UI 定义 — 2x2 网格布局：
  - 第一行：`CandlestickChart`（蜡烛图）+ `OhlcChart`（OHLC 图）
  - 第二行：`LineChart`（折线图）+ `AreaChart`（面积图）
- **第52-56行**：App 结构体
- **第58-59行**：`MatchEvent` — 空实现
- **第62-71行**：`AppMain` 标准实现

## 关键 API

- `CandlestickChart` — 蜡烛图/ K 线图
- `OhlcChart` — OHLC 图
- `LineChart` — 折线图
- `AreaChart` — 面积图

所有图表组件均内置数据生成，无需外部数据源。
