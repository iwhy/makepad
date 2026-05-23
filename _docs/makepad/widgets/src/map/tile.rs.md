# map/tile.rs — 地图瓦片加载与解析

## 整体职能
该文件实现地图瓦片（Tile）的完整生命周期管理：从 HTTP(S) 网络请求或本地 MBTiles 文件读取矢量瓦片数据，经 Gzip 解压、Protocol Buffer (protobuf) 解析后，提取地理要素（道路/建筑/水体等）和标签，并缓存在 LRU 缓存中供渲染使用。

## 核心类型与常量

- **`TileKey`**：瓦片唯一标识，包含 `z`（缩放级别）、`x`（列号）、`y`（行号）三个整数。实现 `Hash + Eq` 用于缓存查找。
- **`TileEntry`**：单个瓦片的运行时状态，包含 `state: TileLoadState`、`last_access`（最近访问时间）、`attempts`（加载尝试次数）。
- **`TileLoadState`**：枚举，取值 `NotLoaded`、`LoadingNetwork`（网络加载中）、`LoadingLocal`（本地加载中）、`Ready { features, labels }`（就绪，包含要素和标签）、`Failed { error }`（加载失败）。
- **`TileFeature`**：单个地理要素，包含 `id`、`source_layer`（源图层名）、`geometry_type`（点/线/面）、`geometry`（编码几何数据）、`attributes`（KV 属性对）。
- **`TileLabel`**：瓦片级标签，包含 `text`、`source_layer`、`priority`、`path_points`（路径点编码）、`road_kind`。
- **`TileCache`**：基于 `HashMap<TileKey, TileEntry>` 的 LRU 缓存，由 `max_entries` 控制大小。
- `DEFAULT_TILE_SIZE`：256（标准瓦片像素尺寸）。
- `MAX_TILE_RETRIES`：3（最大重试次数）。
- `LOCAL_MBTILES_MIN_ZOOM` / `LOCAL_MBTILES_MAX_ZOOM`：本地瓦片的缩放范围。
- `TILE_CACHE_MAX_ENTRIES`：512（缓存最大瓦片数）。

## 方法与实现逻辑

### `fn register_types` — 类型注册
将 `TileKey`、`TileLoadState`、`TileFeature`、`TileCache` 等类型注册到脚本运行时，使瓦片状态可在调试 UI 中查看。

### `fn request_tile` — 请求瓦片
接收 `TileKey`，检查缓存中是否已有：
1. **缓存命中**：更新 `last_access`，返回就绪状态并附加现有数据。
2. **缓存缺失**：创建 `TileEntry`，状态设为 `LoadingLocal`（本地模式）或 `LoadingNetwork`（网络模式），启动异步加载任务。
3. 如果缓存超过 `TILE_CACHE_MAX_ENTRIES`，驱逐最近最少使用的瓦片 (LRU eviction)。

### `fn load_tile_network` — 网络加载瓦片
通过 `HttpClient` 发送 GET 请求到瓦片 URL（URL 模板由 `MapTileSource` 提供，如 `https://tile.example.com/{z}/{x}/{y}.pbf`）：
1. 构建完整 URL（替换 `{z}`、`{x}`、`{y}` 占位符）。
2. 发送异步 HTTP 请求，设置超时（默认 10 秒）。
3. 收到响应后检查 HTTP 状态码。
4. 对响应体进行 Gzip 解压（如果 `Content-Encoding: gzip`）。
5. 调用 `parse_tile_data` 解析 protobuf。
6. 成功后状态改为 `Ready`，失败后递增 `attempts`，未超过重试次数则重新加入加载队列。

### `fn load_tile_local` — 本地 MBTiles 加载
从本地 MBTiles 文件（SQLite 数据库）读取瓦片 blob：
1. 打开 SQLite 连接（或使用已缓存的连接）。
2. 执行查询：`SELECT tile_data FROM tiles WHERE zoom_level=? AND tile_column=? AND tile_row=?`。
3. 处理 TMS 坐标翻转：MBTiles 使用 TMS 规范，Y 轴与 XYZ 规范相反，需做 `tms_y = 2^z - 1 - y` 转换。
4. 对读取的 blob 进行 Gzip 解压。
5. 调用 `parse_tile_data` 解析。

### `fn parse_tile_data` — 解析瓦片数据
将解压后的字节流按 Mapbox Vector Tile (MVT) 格式 v2.1 解析：
1. 使用 `protobuf` 库解码三层嵌套结构：Tile → Layers → Features。
2. 对每个 `Feature`，提取 `id`、`type`（Point / Linestring / Polygon）、`geometry` 命令序列和 `tags`（key-value 对）。
3. 使用 `tile_layer.key_values` 将整数 tag 引用解析为字符串键值对。
4. 对地理要素进行分类：道路要素提取 `label` 信息（道路名称 `name`、道路种类 `highway`、优先级 `priority`）。
5. 返回 `(Vec<TileFeature>, Vec<TileLabel>)` 元组。

### `fn get_visible_tiles` — 计算可见瓦片
根据当前视口中心坐标 `(lat, lng)`、缩放级别 `zoom` 和视口像素尺寸 `(width, height)` 计算需要加载的瓦片列表：
1. 将中心经纬度转换为瓦片坐标 `(tile_x, tile_y)`。
2. 计算视口覆盖的瓦片范围（加上 `TILE_OVERDRAW` 1 圈额外的瓦片以防止边缘闪烁）。
3. 生成 `TileKey` 列表并按从中心到边缘排序，优先级最高的瓦片最先加载。

### `fn tile_key_for_coordinate` — 坐标转 TileKey
将经纬度坐标 `(lat, lng)` 和缩放级别 `z` 转换为 `TileKey`：
```
n = 2^z
xtile = floor((lng + 180) / 360 * n)
ytile = floor((1 - log(tan(lat_rad) + 1/cos(lat_rad)) / π) / 2 * n)
```

### `fn tile_key_to_bbox` — TileKey 转地理边界
返回瓦片覆盖的地理范围 `(west, south, east, north)`，用于瓦片坐标到屏幕坐标的映射计算。

### `fn coordinate_to_screen` — 地理坐标转屏幕坐标
将 `(lat, lng)` 经纬度转换为相对于地图视口的像素坐标 `(x, y)`：
```
x = (lng - center_lng) * cos(center_lat_rad) * meters_per_pixel
y = (lat - center_lat) * meters_per_pixel
```

### `fn clear_cache` — 清除缓存
清除所有瓦片缓存条目。当地图源或样式变更时需要调用。

### `fn cancel_pending_loads` — 取消待加载请求
遍历所有状态为 `LoadingNetwork` 的瓦片条目，取消关联的 HTTP 请求并标记为 `NotLoaded`，释放连接资源。
