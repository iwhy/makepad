# `audio_stream.rs` — 智能音频流缓冲

## 概述
该文件实现了一个带智能缓冲策略的音频流收发通道。核心设计目标是：在网络突发或系统负载波动时保持低延迟播放的同时，通过自适应缓冲避免欠载（underrun）和溢出（overflow）。

### 缓冲策略
- 正常运行时以**最小延迟**输出（不强制定缓冲量）。
- 发生欠载（underrun）时，回退到**缓冲模式**，累积到 `min_buf` 后再恢复输出。
- 发生溢出（达到 `max_buf`）时，丢弃数据恢复到 `min_buf` 重新同步。
- **自适应 `max_buf`**：在网络突发导致频繁溢出时临时增大缓冲区，稳定后再逐步降低。

---

## 核心数据结构

### `AudioStreamSender`
轻量级发送端，内部持有一个 `Sender<(u64, AudioBuffer)>` 通道发送器，通过 `send(route_id, buffer)` 方法将音频数据送入通道。`route_id` 用于区分多路音频流。实现了 `Send` 和 `Clone`，可跨线程发送。

### `ReceiverInner`
接收端内部状态，受 `Mutex` 保护：
- **`routes`**：`Vec<AudioRoute>`，管理所有活跃音频路由。
- **`min_buf` / `max_buf`**：基础缓冲策略参数（以输出 chunk 为单位）。
- **`stream_recv`**：通道接收端，接收 `(u64, AudioBuffer)` 元组。

### `AudioRoute`
每条音频路由的独立状态机：
- **`id`**：路由唯一标识。
- **`start_offset`**：当前首缓冲区内已读取的帧偏移。
- **`buffers`**：`VecDeque<AudioBuffer>` 待播放的音频缓冲区队列。
- **`is_buffering`**：是否处于缓冲模式（欠载后的等待状态）。
- **`min_buf_multiplier` / `max_buf_multiplier`**：自适应乘数，初始为 1，在稳定/波动时动态调整，上限 16（4 倍）。
- **`stable_chunks`**：连续稳定输出的 chunk 计数，用于自适应降级策略。

---

## 关键方法实现

### `AudioStreamSender::create_pair(min_buf, max_buf)`
工厂方法，创建一对 `(AudioStreamSender, AudioStreamReceiver)`。底层通过 `std::sync::mpsc::channel` 创建无界通道，返回发送端和接收端。

### `AudioStreamReceiver::try_recv_stream()`
**非阻塞**接收通道数据。每次调用最多处理 20 个数据包（防止在网络突发时长时间占用音频线程）。新到达的数据包根据 `route_id` 追加到对应路由的缓冲队列尾部，若路由不存在则自动创建新路由。

### `AudioStreamReceiver::recv_stream()`
**阻塞**接收一个数据包，然后调用 `try_recv_stream()` 冲刷剩余的全部待处理数据包（最多 20 个）。适合在音频回调中首次调用来获取数据。

### `AudioStreamReceiver::read_buffer(underrun_ok, route_num, output)`
核心读取逻辑，从指定路由的缓冲区队列中读取数据填充到 `output` 中，返回实际读取的帧数：

1. **计算可用帧数**：累加队列中所有缓冲区的帧数，减去已消费的 `start_offset`。
2. **缓冲模式判断**：若处于 `is_buffering` 状态，检查可用帧是否达到 `chunk_size * min_buf * min_buf_multiplier`，未达到则返回 0（继续缓冲）。
3. **欠载检测**：可用帧不足一个 chunk 时，若 `underrun_ok` 为 false 且稳定计数小于 500，则倍增 `min_buf_multiplier`（最快速度增大缓冲目标值）。进入缓冲模式，重置稳定计数。
4. **溢出检测**：可用帧超过 `chunk_size * effective_max_buf` 时，若突发频繁（stable_chunks <= 500）则倍增 `max_buf_multiplier`；否则将队列数据丢弃到目标水位。
5. **自适应降级**：连续稳定输出超过 500 个 chunk 后，逐步将 `max_buf_multiplier` 和 `min_buf_multiplier` 除以 2 向 1 收敛，回退到低延迟模式。
6. **帧数据复制**：从队列头缓冲区开始，按声道逐帧复制到 output 中。若队列头缓冲区剩余帧不足一帧，则弹出头缓冲区，继续处理下一个。遇到声道数不足的缓冲区时，重复最后一个声道进行填充。
