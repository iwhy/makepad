# ui_signal.rs — UI 事件循环信号集成

**File path**: `platform/network/src/ui_signal.rs` (193 行)
**Core purpose**: 提供网络事件与 Makepad UI 事件循环之间的跨线程信号桥接，使用原子标志位和 mpsc channel 实现非阻塞通知。

## 全局信号

### UI_SIGNAL / ACTION_SIGNAL (`AtomicBool`)
全局静态原子标志，用于唤醒 UI 线程的事件循环。

## 结构体

### SignalToUI
- `inner: Arc<AtomicBool>` — 每个实例可独立检测
- `set()` — 设置自身标志 + 全局 UI_SIGNAL
- `check_and_clear()` — 检查并清除本地标志
- `check_and_clear_ui_signal()` (静态) — 检查并清除全局 UI_SIGNAL
- `check_and_clear_action_signal()` (静态) — 检查并清除全局 ACTION_SIGNAL

### SignalFromUI
- `inner: Arc<AtomicBool>` — UI 到后端的信号
- `set()` — 设置标志（不触发全局信号）
- `check_and_clear()` — 检查并清除

### ToUIReceiver<T> / ToUISender<T>
通过网络后端 → UI 方向发送消息：
- `ToUIReceiver` 持有 `Receiver<T>`，暴露 `try_recv()` 和 `try_recv_flush()`
- `try_recv_flush()` — 清空队列返回最后一条消息（适用于对最新状态感兴趣的场景）
- `ToUISender` 持有 `Sender<T>`，`send()` 调用后自动设置 `SignalToUI::set_ui_signal()` 唤醒 UI

### FromUISender<T> / FromUIReceiver<T>
UI → 网络后端方向：
- `FromUISender` 持有 `Sender<T>` + 可选的 `Receiver<T>`
- `new_channel()` 重建 channel
- `receiver()` 获取对应的 `FromUIReceiver`
- `FromUIReceiver` 实现 `Deref<Target = Receiver<T>>`

## Send 安全
- `ToUISender<T>` 和 `FromUIReceiver<T>` 显式标记为 `Send`（基于 `T: Send`）
