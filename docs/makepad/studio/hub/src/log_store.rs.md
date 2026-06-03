# `log_store.rs` — 日志和性能分析存储

## 文件位置
- 路径: `studio/hub/src/log_store.rs`
- 行数: 258 行
- 作用: 提供线程安全的日志条目存储/查询和性能采样数据管理

## 日志系统 (`LogStore`)

### `AppendLogEntry` (输入结构)
```rust
pub struct AppendLogEntry {
    pub build_id: Option<QueryId>,
    pub level: LogLevel,
    pub source: LogSource,
    pub message: String,
    pub file_name: Option<String>,
    pub line: Option<usize>,
    pub column: Option<usize>,
    pub timestamp: Option<f64>,
}
```

### `LogStore`
```rust
pub struct LogStore {
    entries: Arc<RwLock<Vec<LogEntry>>>,
}
```
- `RwLock` 提供并发读写，多个 reader 可同时访问
- `Arc` 允许跨线程共享

| 方法 | 说明 |
|------|------|
| `append(&mut self, entry)` | 添加日志条目，自动分配 index 和 timestamp（若未提供），返回 `(index, LogEntry)` |
| `clear(&mut self)` | 清除所有条目 |
| `query(&self, query: &LogQuery)` | 返回匹配的 `(index, LogEntry)` 向量 |
| `entries_handle()` | 返回 `Arc<RwLock<Vec<LogEntry>>>` 的克隆，供外部直接遍历 |

### `LogQuery` (查询过滤器)
```rust
pub struct LogQuery {
    pub build_id: Option<QueryId>,
    pub level: Option<String>,
    pub source: Option<LogSource>,
    pub file: Option<String>,
    pub pattern: Option<String>,
    pub since_index: Option<usize>,
}
```
- `matches()` 方法实现 AND 语义：所有指定字段都必须匹配
- `level` 匹配通过 `matches_level()` 函数，接受 `"error"/"Error"`、`"warning"/"Warning"/"warn"/"Warn"`、`"log"/"Log"/"info"/"Info"`
- `pattern` 使用 `message.contains(pattern)`（子串匹配，非正则）

### `fn query_log_entries(entries, query) -> Vec<(usize, LogEntry)>`
纯函数，遍历切片并过滤匹配条目

## 性能分析系统 (`ProfilerStore`)

### `ProfilerStore`
```rust
pub struct ProfilerStore {
    event_samples: Vec<(Option<QueryId>, EventSample)>,
    gpu_samples: Vec<(Option<QueryId>, GPUSample)>,
    gc_samples: Vec<(Option<QueryId>, GCSample)>,
}
```
三种采样类型各自独立存储

| 方法 | 说明 |
|------|------|
| `append_event(build_id, sample)` | 添加 Event 采样 |
| `append_gpu(build_id, sample)` | 添加 GPU 采样 |
| `append_gc(build_id, sample)` | 添加 GC 采样 |
| `query(&self, query)` | 统一查询三种采样，返回 `(events, gpus, gcs, total)` |

### `ProfilerQuery`
```rust
pub struct ProfilerQuery {
    pub build_id: Option<QueryId>,
    pub sample_type: Option<LiveId>,
    pub time_start: Option<f64>,
    pub time_end: Option<f64>,
    pub max_samples: Option<usize>,
}
```
- `sample_type` 使用 `LiveId` 常量识别：`SAMPLE_TYPE_EVENT`, `SAMPLE_TYPE_GPU`, `SAMPLE_TYPE_GC`
- 时间过滤基于 `at` 字段（Unix 时间戳秒数）

### 降采样算法 (`downsample`)
```rust
fn downsample<T: Clone>(items: &[T], max: usize) -> Vec<T>
```
最大输出 `max` 个样本：
- 不超过 `max` → 原样返回
- `max == 1` → 返回第一个
- 通过 `idx = i * (n-1) / (m-1)` 均匀选点（包含首尾）

### 辅助函数

| 函数 | 说明 |
|------|------|
| `now_seconds()` | 返回当前 Unix 时间戳（秒，浮点精度） |
| `query_profiler_sample()` | 单条采样的匹配函数 |

## 设计观察

- 日志存储和 Profiler 存储独立，分别服务于 Studio 的 Log 和 Profiler 面板
- 降采样保证前端性能可视化不会因为数据太多而卡顿
- `RwLock` 写锁在 append/clear 时获取，读取查询可并发
- 日志按索引线性增长，不支持删除单条
