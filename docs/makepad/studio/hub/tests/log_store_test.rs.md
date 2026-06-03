# log_store_test.rs

## 概述

测试 `LogStore` 和 `ProfilerStore` 的日志存储、过滤和降采样功能。这两个 store 是 Hub 内部管理构建日志和性能分析样本的核心组件。

## 测试场景

### `log_store_filters_by_build_source_and_pattern`

- **场景**: 查询日志时需要按 `build_id`、日志级别、来源、文件名和内容模式组合过滤。
- **流程**:
  1. 创建两个不同 `build_id` 的日志条目（`build_a` 的 Cargo 信息日志，`build_b` 的 Studio 警告日志）。
  2. 构建 `LogQuery`，组合 `build_id`、`level`、`source`、`file` 和 `pattern` 条件。
  3. 执行查询并断言仅返回匹配条目（1 条记录，ID 为 1）。
- **验证重点**:
  - 多条件组合过滤的正确性。
  - `since_index` 偏移查询。

### `profiler_store_filters_and_downsamples`

- **场景**: ProfilerStore 需要支持按时间范围过滤和最大样本数降采样。
- **流程**:
  1. 为同一 `build_id` 插入 20 个 `EventSample` 和 10 个 `GPUSample`。
  2. 执行事件查询：时间范围 [4.0, 15.0)，`max_samples` 限制为 5。
  3. 断言：事件样本数 <= 5，GPU 样本为空，`total` 计数为 12（时间范围内共 12 个事件）。
  4. 执行 GPU 查询：不加时间限制，`max_samples` 为 3。
  5. 断言：GPU 样本数为 3，`total` 计数为 10。
- **验证重点**:
  - 时间窗口过滤。
  - `max_samples` 降采样上限。
  - `sample_type` 区分事件/GPU/GC 样本。
  - 查询结果返回 `(events, gpu, gc, total)` 四元组。

## 测试模式

- **纯单元测试**：不启动 Hub 后端，直接实例化 store 进行测试。
- **业务规则验证**：测试日志存储引擎的过滤、采样和查询语义。
