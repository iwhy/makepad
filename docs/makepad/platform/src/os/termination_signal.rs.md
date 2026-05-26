# `termination_signal.rs` — 进程终止信号处理

## 文件定位

此文件为 Makepad 桌面应用提供优雅的 Ctrl+C（SIGINT）信号处理机制。它使用 `ctrlc` 第三方 crate 注册一个全局信号处理器，将终止请求以原子标志的方式通知给主事件循环，而非直接 `exit` 从而允许应用清理资源。

---

## 全局状态

```rust
static INSTALLED: AtomicBool = AtomicBool::new(false);
static REQUESTED: AtomicBool = AtomicBool::new(false);
```

- **`INSTALLED`**：标记信号处理器是否已注册。防止重复注册或跨线程竞态注册。
- **`REQUESTED`**：标记用户是否发起了终止请求。信号处理器收到 Ctrl+C 时置为 `true`，主循环检测到后执行退出流程。
- 两个变量都是 `AtomicBool`，确保跨线程安全访问。`INSTALLED` 只在初始化时写入，`REQUESTED` 在信号处理器（任意线程）和主循环（通常是 UI 线程）之间共享。

---

## `install()`

```rust
pub(crate) fn install() {
    if INSTALLED.swap(true, Ordering::AcqRel) {
        return;
    }
    if let Err(err) = ctrlc::set_handler(move || {
        REQUESTED.store(true, Ordering::Release);
        SignalToUI::set_ui_signal();
    }) {
        log!("Failed to install termination signal handler: {err}");
    }
}
```

- **职责**：注册进程终止信号（Ctrl+C/SIGINT）处理器。
- **实现逻辑**：
  1. 使用 `INSTALLED.swap(true, Ordering::AcqRel)` 进行原子的"检测并设置"。如果返回 `true` 表示已经注册过，直接返回。
  2. 调用 `ctrlc::set_handler` 注册信号处理器闭包。
  3. 信号处理器闭包执行两个操作：
     a. `REQUESTED.store(true, Ordering::Release)`：设置终止请求标志。使用 `Release` 内存序保证之前的所有写操作在 `REQUESTED` 对其他线程可见之前已完成。
     b. `SignalToUI::set_ui_signal()`：通知 UI 事件循环有新信号需要处理。这通常会唤醒阻塞等待事件的 UI 线程（例如 epoll/kqueue 上的事件循环）。
  4. 如果 `ctrlc::set_handler` 注册失败（例如系统不支持或已经有过信号处理器），记录错误但不 panic——应用继续运行，只是 Ctrl+C 不会触发优雅退出。
- **设计要点**：
  - 使用 `AcqRel` 内存序确保 `INSTALLED` 的读写在多线程间正确同步。
  - `INSTALLED.swap` 代替 `compare_exchange`，因为我们不需要知道旧值，只需要保证只执行一次。
  - 不直接在信号处理器中执行复杂逻辑（如调用 `exit` 或释放资源），只设置标志并通过 `SignalToUI` 唤醒主线程。这符合信号处理的黄金法则——信号处理器中做最少的事情。

---

## `take_requested()`

```rust
pub(crate) fn take_requested() -> bool {
    REQUESTED.swap(false, Ordering::AcqRel)
}
```

- **职责**：检查并消费终止请求标志。如果返回 `true`，调用者应执行退出流程。
- **实现逻辑**：
  1. 使用 `REQUESTED.swap(false, Ordering::AcqRel)` 将标志重置为 `false`。
  2. 返回旧的标志值。
  3. 使用 `AcqRel` 保证：读取到已被信号处理器写入的 `true` 时，能同时看到信号处理器在 `RELEASE` 之前的所有操作。
- **用途**：在主事件循环中定期检查，检测到 `true` 时触发 `Event::Shutdown` 或直接退出循环。
- **设计要点**：
  - `swap`（而非 `load`）是"take"语义——消费掉请求，防止同一请求被处理两次。
  - 如果信号处理器在 `take_requested` 之后再次触发，`REQUESTED` 会再次变为 `true`，下一次调用时返回 `true`。但通常 Ctrl+C 只触发一次。

---

## `SignalToUI` 交互

此处使用了 `crate::thread::SignalToUI::set_ui_signal()`。这是一个跨平台信号唤醒机制，其实现可能包括：

- **Linux**：向事件循环的 eventfd 写入一个字节。
- **macOS**：向 `CFRunLoopSource` 发送信号或使用 `mach_port`。
- **Windows**：使用 `PostThreadMessage` 或 `SetEvent`。

核心目的是打破主线程的事件等待状态，使其立刻检查 `REQUESTED` 标志。

---

## 使用模式

```rust
// 初始化（应用启动时）
crate::os::termination_signal::install();

// 主事件循环中
if crate::os::termination_signal::take_requested() {
    log!("Termination signal received, shutting down");
    break;  // 退出事件循环
}
```

---

## 总结

`termination_signal.rs` 虽然只有 24 行，但实现了一个完整的最小化优雅退出机制：

| 状态 | 原子变量 | 生命周期 |
|------|---------|---------|
| 是否已安装 | `INSTALLED` | 整个进程生命周期（只写一次） |
| 是否请求终止 | `REQUESTED` | 从信号发起到主线程消费（短暂） |

信号安全（signal-safe）做法：
1. 不在信号处理器中分配内存。
2. 不在信号处理器中调用非 async-signal-safe 函数。
3. 只做两件事：写原子变量、唤醒 UI 线程。
