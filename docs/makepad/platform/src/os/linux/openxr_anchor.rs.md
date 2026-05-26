# openxr_anchor.rs

One-liner (EN): OpenXR spatial anchor management — creation, persistence, querying, and colocation of local/cloud spatial anchors for XR experiences.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/openxr_anchor.rs` (938 行)
- **核心作用**: 实现 OpenXR 空间锚点（Spatial Anchor）的完整生命周期管理：创建本地锚点、持久化到磁盘/云、查询恢复、空间组件状态设置以及协同定位（colocation）发现与广告。

## 类型/结构体

### `CxOpenXrAnchor` — 锚点状态容器

| 字段 | 类型 | 说明 |
|------|------|------|
| `anchor` | `Option<SpaceAnchor>` | 当前空间锚点（左右手各一个 XrSpace） |
| `async_state` | `AsyncAnchorState` | 异步操作状态机 |
| `persisted_anchor_present` | `bool` | 磁盘上有持久化锚点 |
| `floor_y` | `Option<f32>` | 缓存的地面 Y 坐标 |

### `SpaceAnchor` — 内部空间锚点

| 字段 | 类型 | 说明 |
|------|------|------|
| `left_space` | `XrSpace` | 左手控制器对应空间 |
| `right_space` | `XrSpace` | 右手控制器对应空间 |

### `AsyncAnchorState` — 异步操作状态机枚举

| 变体 | 说明 |
|------|------|
| `Idle` | 空闲状态 |
| `GetLocalAnchor { request_id, left_uuid, right_uuid }` | 正在从本地存储按 UUID 查询锚点 |
| `ClearAndSet { request_id, left_pose, right_pose }` | 清除旧锚点后新建 |
| `LocalAnchorLeft { request_id, right_pose }` | 已在创建左手锚点，等待完成 |
| `LocalAnchorRight { request_id, left_space }` | 左手锚点已创建，正在创建右手锚点 |
| `LocalAnchorSaveSpaces { request_id }` | 正在保存锚点到空间存储 |
| `LocalAnchorShareSpaces { request_id }` | 正在共享锚点到协同空间 |

### `AnchorAdvertisement` — 协同定位广告数据

```rust
#[derive(SerBin, DeBin)]
struct AnchorAdvertisement {
    group_uuid: XrUuid,
    anchor_left_uuid: XrUuid,
    anchor_right_uuid: XrUuid,
}
```

## 关键方法

### CxOpenXr 方法

| 方法 | 说明 |
|------|------|
| `advertise_anchor(anchor)` | 创建可共享锚点（预留，未完全实现） |
| `set_local_anchor(anchor)` | **设置本地锚点**：启动异步创建流程（左手→右手→保存） |
| `set_local_floor(floor_y, os_type)` | 设置并持久化地面 Y 坐标 |
| `get_local_anchor(os_type)` | **获取本地锚点**：从缓存文件读取 UUID 并发起空间查询 |
| `discover_anchor(id)` | 发现锚点（预留，未完全实现） |

### CxOpenXrSession 方法

#### 事件处理
| 方法 | 说明 |
|------|------|
| `handle_anchor_events(xr, event_buffer, os_type)` | 分发所有锚点相关 XR 事件 |

处理的事件类型:
- `EVENT_DATA_SPATIAL_ANCHOR_CREATE_COMPLETE_FB` — 锚点创建完成回调
- `EVENT_DATA_SPACE_SET_STATUS_COMPLETE_FB` — 空间组件状态设置完成
- `EVENT_DATA_SHARE_SPACES_COMPLETE_META` — 空间共享完成
- `EVENT_DATA_START_COLOCATION_DISCOVERY_COMPLETE_META` — 开始协同定位发现完成
- `EVENT_DATA_COLOCATION_DISCOVERY_RESULT_META` — 发现结果
- `EVENT_DATA_COLOCATION_DISCOVERY_COMPLETE_META` — 发现完成
- `EVENT_DATA_START/STOP_COLOCATION_ADVERTISEMENT_COMPLETE_META` — 广告开始/停止完成
- `EVENT_DATA_SPACE_QUERY_RESULTS_AVAILABLE_FB` — 空间查询结果可用
- `EVENT_DATA_SPACES_SAVE/ERASE_RESULT_META` — 空间保存/擦除结果

#### 锚点操作
| 方法 | 说明 |
|------|------|
| `set_local_anchor(xr, anchor)` | 设置本地锚点的异步流程起点 |
| `set_local_floor(floor_y, os_type)` | 设置地面高度并缓存 |
| `get_local_anchor(xr, os_type)` | 从缓存恢复锚点 |
| `create_anchor_request(xr, pose)` | 调用 `xrCreateSpatialAnchorFB` 发起创建请求 |
| `create_anchor_response(xr, response, os_type)` | 处理创建响应，按状态机推进（左手→右手→保存） |
| `set_space_status_request(xr, spaces, flags)` | 枚举并设置空间组件状态（STORABLE, LOCATABLE 等） |
| `save_spaces_request(xr, spaces)` | 保存空间到持久化存储 |
| `save_spaces_response(xr, response)` | 处理保存响应 |
| `query_local_spaces_request(xr, uuids)` | 按 UUID 本地查询空间 |
| `query_spaces_response(xr, response)` | 处理查询响应，恢复 `SpaceAnchor` |
| `erase_spaces_request(xr, spaces)` | 擦除空间 |
| `share_spaces_request/response` | 空间共享（暂未启用） |
| `start/stop_colocation_advertisement_request` | 协同定位广告（暂未启用） |
| `start/stop_colocation_discovery_request` | 协同定位发现（暂未启用） |

### CxOpenXrAnchor 方法

| 方法 | 说明 |
|------|------|
| `anchor_persisted() -> bool` | 是否有可用锚点（内存或磁盘） |
| `floor_y() -> Option<f32>` | 获取地面 Y 坐标 |
| `set_floor_y(os_type, floor_y)` | 设置地面 Y 坐标并缓存到文件 |
| `locate_anchor(xr, local_space, predicted_display_time) -> Option<XrAnchor>` | 定位当前锚点：从左右手的 XrSpace 获取位置 |

## 持久化缓存

| 缓存文件 | 内容 |
|----------|------|
| `left_anchor` | 左手锚点的 UUID（16 字节二进制） |
| `right_anchor` | 右手锚点的 UUID（16 字节二进制） |
| `floor_y` | 地面 Y 坐标（文本格式） |

缓存目录通过 `OsType::get_cache_dir()` 获取。

## 实现细节

### 异步状态机（设置锚点流程）

```
set_local_anchor()
  │
  ▼
AsyncAnchorState::LocalAnchorLeft
  │  └ create_anchor_request() → xrCreateSpatialAnchorFB
  │
  ▼ [EVENT: CreateComplete]
create_anchor_response()
  │  └ 写入 left_uuid 缓存
  │  └ create_anchor_request(right_pose)
  ▼
AsyncAnchorState::LocalAnchorRight
  │
  ▼ [EVENT: CreateComplete]
create_anchor_response()
  │  └ 写入 right_uuid 缓存
  │  └ set_space_status_request(STORABLE)
  │  └ save_spaces_request()
  ▼
AsyncAnchorState::LocalAnchorSaveSpaces
  │
  ▼ [EVENT: SpacesSaveResult]
save_spaces_response()
  ▼
AsyncAnchorState::Idle ✓
```

### 查询恢复流程

```
get_local_anchor()
  │  └ 从缓存读取 UUID
  │  └ query_local_spaces_request()
  ▼
AsyncAnchorState::GetLocalAnchor
  │
  ▼ [EVENT: QueryResultsAvailable]
query_spaces_response()
  │  └ 按 UUID 匹配左右空间
  │  └ set_space_status_request(STORABLE, LOCATABLE)
  ▼
SpaceAnchor { left_space, right_space } ✓
```

- 所有 XR 函数调用使用 `.to_result()` / `.log_error()` 扩展方法进行结果处理
- 状态机错误通过 `AsyncAnchorState::error()` 方法处理，发生错误时回退到 `Idle`
- 使用 `xr_array_fetch` 辅助宏枚举空间支持的组件类型
- `locate_anchor()` 调用 `XrSpaceLocation::locate()` 获取实时位姿
