# select_timer.rs — 基于 select() 的定时器管理

**文件路径**: `platform/src/os/linux/select_timer.rs` (171 行)
**核心功能**: 使用 POSIX `select()` 系统调用实现的高效定时器管理，支持单次和重复定时器。

## 主要类型

### `SelectTimer`
单个定时器条目：
- `id: u64` — 定时器标识
- `timeout: f64` — 原始超时时间（秒）
- `repeats: bool` — 是否重复
- `delta_timeout: f64` — 相对前一个定时器的增量超时

### `SelectTimers`
有序定时器队列管理器：
- `timers: VecDeque<SelectTimer>` — 按触发时间排序的定时器列表
- `time_start: Instant` — 基准时间
- `select_time: f64` — 上次 `select` 调用的时间

## 关键方法

### `SelectTimers::select(fd)`
执行 `select()` 系统调用：
1. 设置文件描述符集合（fd 0 和用户指定的 fd）
2. 将队首定时器的 `delta_timeout` 转换为 `timeval` 结构体
3. 调用 `select` 等待 I/O 事件或超时

### `SelectTimers::update_timers(out)`
更新定时器状态并收集已触发的定时器：
1. 计算自上次 `select` 以来的经过时间
2. 遍历定时器队列，将经过时间与各定时器的 `delta_timeout` 比较
3. 当 `select_time_used >= delta_timeout` 时触发该定时器
4. 重复定时器自动重新加入队列
5. 收集触发的定时器 ID 到输出向量

### `SelectTimers::start_timer(id, timeout, repeats)`
启动定时器：
1. 计算定时器在有序队列中的插入位置
2. 初始化 `delta_timeout` 为完整超时时间
3. 从前面定时器的 `delta_timeout` 中减去已占用的时间
4. 调整后一个定时器的 `delta_timeout`

### `SelectTimers::stop_timer(id)`
停止定时器：
1. 在队列中查找指定 ID 的定时器
2. 将其 `delta_timeout` 累加到后一个定时器
3. 从队列中移除

## 实现细节

- 使用差量超时（delta timeout）链表设计，每个定时器只存储与前一个的相对时间
- 这样 `select` 的超时值总是队首定时器的 `delta_timeout`
- 时间复杂度：`start_timer` 为 O(n)，`update_timers` 为 O(k)（k 为触发的定时器数）
