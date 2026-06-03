# map/src/main.rs

演示 Makepad 的 `MapView` 地图组件。

## 整体结构

- **第7-24行**：`script_mod!` UI 定义 — 窗口中使用 `MapView` 组件，Fill/Fill 尺寸填满窗口
- **第26-30行**：App 结构体
- **第32-34行**：`MatchEvent` — 空实现，无额外事件处理
- **第36-46行**：`AppMain` 实现

## 关键 API

- `MapView` — 嵌入式地图渲染 widget，支持交互操作（平移、缩放）
- 只需在 DSL 中声明即可工作，无需 Rust 侧额外事件处理
