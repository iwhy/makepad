# `performance_stats.rs` — 性能统计

## 概述

`PerformanceStats` 跟踪帧渲染性能数据，记录最近一段时间内周期最长的帧耗时。它使用一个固定大小的滑动窗口（最多 100 条记录）来存储每个 100ms 周期内的最大帧时间，用于性能监控和诊断 UI。

## 核心类型

### `struct FrameStats`
单条帧统计记录：
- **`occurred_at: f64`**：该帧发生的时间戳（秒），作为记录的主键。
- **`time_spent: f64`**：该帧的渲染耗时（秒）。

### `struct PerformanceStats`
- **`last_frame_time: Option<f64>`**：上一帧的时间戳，用于计算帧间隔。
- **`max_frame_times: VecDeque<FrameStats>`**：固定容量的双端队列，按时间倒序（最新在前）存储帧统计信息。

`Default` 初始化时 `last_frame_time` 为 `None`，队列预留容量 100。

## 方法

### `process_frame_data(time: f64)`
处理新一帧的时间数据。实现逻辑：
1. 检查是否有 `last_frame_time`：
   - 若为 `None`（首帧），仅记录当前时间并返回。
2. 计算当前帧耗时 `time - previous_time`。
3. 检查队列是否为空：
   - 为空时直接插入一条 `FrameStats` 记录，返回。
4. 计算当前时间所在的 100ms 周期标识：`(time * 10.0) as i64`（即 `floor(time * 10)`）。与之对比队首（最新）记录的 `occurred_at` 周期标识：
   - **属于同一周期**：若新帧耗时大于队首记录的值，则更新队首记录（保留周期内的最大帧耗时）。
   - **属于新周期**：若队列已满（>=100），从队尾弹出最旧的记录；然后在队首插入新的 `FrameStats` 记录。
5. 更新 `last_frame_time` 为当前时间。
