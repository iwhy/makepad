# map/mod.rs — 地图模块入口

## 整体职能
`map` 模块实现了 **矢量地图渲染引擎**，支持从本地 MBTiles 文件或远程 TileJSON 服务加载地图数据，渲染道路、建筑、水体等地理要素，并为道路名称提供沿路径的文本标注。

## 模块结构
该模块包含以下子模块：

| 文件 | 职责 |
|------|------|
| `mod.rs` | 模块入口，重新导出所有公共类型 |
| `view.rs` | `MapView` widget 主实现，负责瓦片管理、视口变换、绘制调度、标签碰撞处理 |
| `geometry.rs` | 几何数据处理：Ramer-Douglas-Peucker 简化、谢尔宾斯基三角剖分 (ear-clipping)、向量瓦片解码、屏幕空间线/面绘制 |
| `label.rs` | 路径标注：沿道路曲线排列文本、碰撞检测 (quadtree)、去重、字体缩放与渲染 |
| `style.rs` | 地图样式系统：从 `MapStyle` 定义编译为按层分组的 `DrawStyle`、主题色解析、缩放可见范围 |
| `tile.rs` | 瓦片生命周期管理：HTTP(S) 网络加载、本地 MBTiles 读取、Gzip 解压、Protocol Buffer 解析、LRU 缓存 |

## 公开类型
- **`MapView`**：顶层地图 widget。
- **`MapTileSource`**：瓦片数据源配置（网络 URL / 本地 MBTiles 路径）。
- **`MapStyleConfig`**：主题样式配置（浅色/深色/自定义）。
- **`MapViewAction`**：地图 Action 枚举，包括 `ZoomChanged`、`PanChanged`、`CenterChanged`、`MapClicked(coord)`、`MapLongPressed(coord)`。

## 方法与实现逻辑

### `pub fn script_mod` — 模块初始化
依次调用 `view::register_widget`、`tile::register_types`、`label::register_types`、`style::register_types` 和 `geometry::register_types` 将所有地图相关类型注册到脚本运行时。这个顺序确保 widget 的依赖项（如 Draw shader 和样式类型）在 `MapView` 之前可用。
