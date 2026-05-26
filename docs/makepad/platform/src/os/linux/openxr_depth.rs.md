# openxr_depth.rs

One-liner (EN): OpenXR environment depth mesh pipeline — reads GPU depth buffers, preprocesses them into TSDF volumes, and publishes height maps for spatial understanding.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/openxr_depth.rs` (400 行)
- **核心作用**: 实现 OpenXR 环境深度数据的全处理管线：从 GPU 读回深度图像 → 预处理为深度网格作业 → 集成到 TSDF（截断符号距离函数）体素场 → 生成投影像高场和表面网格。运行在独立后台线程中。

## 类型/结构体

### `CxOpenXrDepthMeshPipeline` — 深度网格管线

| 字段 | 类型 | 说明 |
|------|------|------|
| `mailbox` | `SharedLatestDepthJobMailbox` | 共享邮箱（Mutex + Condvar），用于从渲染线程向工作线程传递深度作业 |
| `store` | `XrTsdfStore` | TSDF 存储（线程安全，管理体素场状态） |
| `next_generation` | `u64` | 下一代作业序号 |
| `last_reset_generation` | `u64` | 上次重置的代数 |
| `last_depth_readback_at` | `Option<Instant>` | 上次深度读回时间戳 |

### `LatestDepthJobMailbox` — 深度作业邮箱

| 字段 | 说明 |
|------|------|
| `latest: Option<DepthMeshJob>` | 最新的深度网格作业 |

线程安全类型：`Arc<(Mutex<LatestDepthJobMailbox>, Condvar)>`

### `PendingDepthCandidate` — 待处理的深度候选项

| 字段 | 说明 |
|------|------|
| `job: DepthMeshJob` | 深度网格作业 |
| `novelty: DepthFrameNovetly` | 新颖度评分 |

## 关键方法

### CxOpenXrDepthMeshPipeline

| 方法 | 说明 |
|------|------|
| `new() -> Self` | 创建管线，启动 `depth_preprocess_tsdf_writer_worker` 后台线程 |
| `submit(vulkan, render_targets, frame, depth_image_index) -> Result<(), String>` | 提交深度帧：验证相机变换 → 检查是否需要读回（节流）→ 从 GPU 读回深度图像 → 构造 `DepthMeshJob` → 通过邮箱发送给工作线程 |

### 辅助函数

| 函数 | 说明 |
|------|------|
| `replace_latest_depth_job(mailbox, job) -> bool` | 替换邮箱中的最新作业（原子操作 + condvar 通知） |
| `take_latest_depth_job(mailbox, timeout) -> Option<DepthMeshJob>` | 从邮箱取出最新作业（带超时等待） |
| `pending_depth_candidate_should_replace(pending, candidate) -> bool` | 判断新候选项是否应替换待处理项：基于新颖度评分、有效样本数、代数的综合比较 |
| `enqueue_pending_depth_candidate(pending, candidate, store)` | 入队待处理候选项（若替换则记录 drop） |
| `matched_height_map_publish_interval(store) -> Duration` | 计算自适应高度图发布间隔（基于协同步骤周期时长） |

### `depth_preprocess_tsdf_writer_worker(mailbox, store)` — 后台工作线程

这是整个深度管线的核心工作线程函数，在无限循环中执行：

**步骤循环**:
1. **检查体素尺寸/重置代数变化**：若配置变化，重建 `DepthMeshVolume`
2. **从邮箱获取最新作业**：超时等待 8ms，批量消费所有待处理作业
3. **新颖度评分与候选筛选**：对每个作业进行新颖度评分，保留最优候选
4. **TSDF 集成**：若节流允许，执行 `preprocess_depth_mesh` → `apply_preprocessed_depth_mesh`
5. **投影像高场处理**：
   - 若启用地表分析：同步布局/玩家裁切，使用刷新预算处理切片
   - 若禁用：清空发布状态
6. **发布高度图**：按间隔发布更新的高度图
7. **发布 TSDF 快照**：若应用了更新或高度图变化，发布新快照并触发 `SignalToUI::set_ui_signal()`
8. **协同步骤**：以 8ms 快速间隔或 33ms 空闲间隔执行 `run_cooperative_step`

## 实现细节

### 管线架构

```
渲染线程 (GPU)                   工作线程 (CPU)
  │                                  │
  │ submit()                         │
  │  ├ 验证相机参数                   │
  │  ├ 节流检查                      │
  │  ├ read_openxr_depth_image()     │
  │  └ replace_latest_depth_job()    │
  │       │                          │
  │       ▼ (mailbox)                │
  │                                  │ depth_preprocess_tsdf_writer_worker()
  │                                  │  ├ take_latest_depth_job()
  │                                  │  ├ score_depth_job_novelty()
  │                                  │  ├ preprocess_depth_mesh()
  │                                  │  ├ apply_preprocessed_depth_mesh()
  │                                  │  ├ TSDF 体素集成
  │                                  │  ├ 投影像高场生成
  │                                  │  └ publish_tsdf_snapshot()
  │                                  │       │
  │                                  │       ▼ SignalToUI::set_ui_signal()
```

### 深度作业邮箱通信

- **线程安全**: `Arc<(Mutex<LatestDepthJobMailbox>, Condvar)>` — Mutex 保护共享数据，Condvar 实现等待/通知
- **`replace_latest_depth_job`**: 原子替换 + condvar 通知（生产者，渲染线程）
- **`take_latest_depth_job`**: 带超时的条件等待 + 取值（消费者，工作线程）
- 工作线程在 8ms 空闲等待后会批量消费所有待处理作业，只保留最优候选进行 TSDF 集成

### 候选筛选策略

`pending_depth_candidate_should_replace` 比较两个候选：
1. 新颖度评分高者优先（分数差 > 1e-4）
2. 分数相同时，有效样本数多者优先
3. 若仍相同，高代数优先

### 节流控制

| 参数 | 默认 | 说明 |
|------|------|------|
| `DEPTH_TSDF_INPUT_THROTTLING_DISABLED` | `true` | TSDF 输入节流全局禁用 |
| `DEPTH_TSD_TARGET_INTEGRATION_INTERVAL_MILLIS` | 外部定义 | 目标集成间隔 |
| `DEPTH_PROJECTED_HEIGHT_REFRESH_INTERVAL_MILLIS` | 外部定义 | 投影像高刷新间隔 |
| `DEPTH_PUBLISHED_HEIGHT_MAP_INTERVAL_MILLIS` | 外部定义 | 高度图发布间隔 |
| `DEPTH_ALIGN_PROJECTED_HEIGHT_MAX_SLICE_CREDITS` | 外部定义 | 最大切片积分 |

- `DEPTH_SURFACE_MESH_IDLE_WAIT_MILLIS = 8` — 工作线程空闲等待时间
- `DEPTH_COOPERATIVE_STEP_INTERVAL_MILLIS = 8` — 快速协同步进间隔
- `DEPTH_COOPERATIVE_IDLE_POLL_INTERVAL_MILLIS = 33` — 空闲轮询间隔（~30 FPS）

### 重置机制

当调用 `store.reset()` 时，`reset_generation` 递增。工作线程检测到变化后重建整个 `DepthMeshVolume`，重置所有状态。这用于清除无效的 TSDF 数据或响应空间变化。
