# `worker_pool.rs` — 泛型线程池

## 文件位置
- 路径: `studio/hub/src/worker_pool.rs`
- 行数: 69 行
- 作用: 提供固定大小的工作线程池，用于分发后台任务

## 核心数据结构

### `WorkerMessage` (内部枚举)
```rust
enum WorkerMessage {
    Job(Box<dyn FnOnce() + Send + 'static>),
    Shutdown,
}
```
- 使用 `dyn FnOnce` 类型擦除闭包，支持任意任务
- Shutdown 信号用于优雅关闭

### `WorkerPool`
```rust
pub struct WorkerPool {
    sender: Sender<WorkerMessage>,
    workers: Vec<JoinHandle<()>>,
    worker_count: usize,
}
```

## 构造函数

### `fn new(worker_count: usize) -> Self`
- 强制最小线程数为 1 (`worker_count.max(1)`)
- 创建 `mpsc::channel` 作为任务队列
- 用 `Arc<Mutex<Receiver>>` 包装让多线程安全共享
- 每个工作线程循环：加锁取消息 → 执行 Job → 收到 Shutdown 或通道断开则退出

## 核心方法

### `fn execute<F>(&self, job: F) where F: FnOnce() + Send + 'static`
- 将闭包装箱后发送到通道
- 忽略 `SendError`（即池已关闭时不 panic）

### `fn worker_count(&self) -> usize`
返回工作线程数

## Drop 实现
1. 向每个工作线程发送一个 Shutdown 信号
2. 依次 `join` 所有线程句柄
3. 确保所有任务完成或线程退出后才释放资源

## 设计观察

- 使用 `Arc<Mutex<Receiver>>` 而非 `mpsc::Receiver` 的多消费者封装
- 没有取消机制：已提交的 Job 一定会被执行（除非通道关闭）
- 没有 backpressure：`execute` 总是立即成功返回（通道有界？不，mpsc channel 默认无界）
- 简单的火-忘模式，适合日志、构建通知等不关心返回值的任务
