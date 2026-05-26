# `web_socket.rs` — Studio WebSocket 通信与远程控制桥接

## 文件定位

该文件实现了 Makepad 框架与 Makepad Studio IDE 之间的 WebSocket 通信通道。它是**远程开发/调试基础设施的核心组件**，允许 Studio 通过 WebSocket 协议实时监控和控制运行中的应用实例。文件内定义了全局状态变量、消息收发的线程模型、协议解析逻辑以及与 `Cx` 应用上下文的集成方法。核心依赖：`makepad_network`（网络运行时）、`makepad_studio_protocol`（协议序列化）、`thread`（UI 线程信号）。

---

## 全局状态变量

### `STUDIO_WEB_SOCKET_THREAD_SENDER: Mutex<Option<Sender<StudioWebSocketThreadMsg>>>`
管理后台线程的通道发送端。后台线程负责批量打包和发送 `AppToStudio` 消息。主线程通过此 sender 发送 `AppToStudio` 数据包和 `Terminate` 信号。

### `STUDIO_NET_RUNTIME: Mutex<Option<Arc<NetworkRuntime>>>`
持有网络运行时的 `Arc` 引用，供后台线程在发送二进制 WebSocket 消息时使用。`Arc` 允许多线程共享同一 `NetworkRuntime`。

### `HAS_STUDIO_WEB_SOCKET: AtomicBool`
标记协议栈是否已启用（无论连接是否成功建立）。设置为 `true` 后，框架内部组件（如截图、widget dump、事件分析）会开始工作。

### `STUDIO_WEB_SOCKET_CONNECTED: AtomicBool`
标记 WebSocket 连接是否已成功建立。在 `WsOpened` 响应消息到达后被设置为 `true`，在 `WsClosed` 或 `WsError` 时被重置为 `false`。

### `STUDIO_STDOUT_MODE: AtomicBool`
启用 stdout 模式。此模式下，`send_studio_message` 将 JSON 行写入标准输出而不是 WebSocket。该模式在无网络环境（如 CI）或需要管道到外部工具时使用。

### `CONTROL_CHANNEL: Mutex<Option<Receiver<StudioToApp>>>`
控制通道接收端。外部代码可通过 `set_control_channel` 注入一个通道，`StudioToApp` 消息将通过此通道传输并由事件循环消费。

### `LOCAL_PROFILE_SAMPLES: Mutex<Vec<LocalProfileSample>>`
本地性能分析样本缓冲区。可累积最多 16,384 个样本，超过时丢弃最早的样本。

---

## `StudioWebSocketThreadMsg` 枚举

```rust
enum StudioWebSocketThreadMsg {
    AppToStudio { message: AppToStudio },
    Terminate,
}
```

后台线程的消息类型：`AppToStudio` 携带应用→Studio 的消息；`Terminate` 信号线程退出。

---

## 核心函数

### `consume_studio_socket_response(response) -> Option<Vec<StudioToApp>>`
**协议分发核心函数**。接收 `NetworkResponse`，判断是否为 Studio WebSocket（`socket_id == STUDIO_SOCKET_ID = 0`），并根据消息类型进行分类处理：
- **`WsOpened`**: 设置 `STUDIO_WEB_SOCKET_CONNECTED = true`，返回空 `Vec`（连接事件无需具体消息）。
- **`WsClosed`**: 设置连接标志为 `false`，记录警告日志，返回空 `Vec`。
- **`WsError`**: 类似 `WsClosed`，但记录错误级别日志。包含具体错误信息字符串。
- **`WsMessage`**: 核心处理路径：
  - **Binary 模式**: 优先尝试反序列化为 `StudioToAppVec`（批量消息向量）。若失败，尝试反序列化为 `HubToClient`，仅处理 `Hello`（忽略其他 Hub 协议消息），其他情况记录警告。
  - **Text 模式**: 若内容为空，忽略。否则尝试 JSON 反序列化为单个 `StudioToApp`。非法 JSON 内容记录警告。
- **非 Studio 套接字消息**: 返回 `None`，由调用者（`recv_studio_websocket_message` 或外部事件处理器）按正常网络事件处理。

### `recv_studio_thread_msg(rx, timeout) -> Result<StudioWebSocketThreadMsg, RecvTimeoutError>`
跨平台的消息接收函数：
- **非 WASM 平台**: 直接委托给 `rx.recv_timeout(timeout)`，由操作系统提供超时支持。
- **WASM 平台**: WASM 不支持 `recv_timeout`，使用**忙等待轮询**实现：在超时时间内循环调用 `try_recv()`，每次失败后调用 `std::thread::yield_now()` 让出 CPU。特殊情况：`timeout == Duration::MAX` 时，使用阻塞的 `recv()` 等待（WASM 单线程模型中此场景受限）。

### `studio_ws_send_binary(data: Vec<u8>) -> Result<(), ()>`
后台线程发送二进制消息的辅助函数。从 `STUDIO_NET_RUNTIME` 获取 `NetworkRuntime` 的克隆，调用 `ws_send` 发送数据。失败时返回 `Err(())`。使用 `LiveId(STUDIO_SOCKET_ID)` 标识目标套接字。

---

## `Cx` 方法

### `has_studio_web_socket() -> bool`
静态方法：检查是否已启用 Studio WebSocket 协议栈（无论连接状态）。用于模块的条件性功能开关。

### `has_studio_web_socket_connected() -> bool`
静态方法：检查 WebSocket 连接是否已成功建立。用于 UI 显示连接状态。

### `set_studio_stdout_mode(enabled: bool)`
启用或禁用 stdout 输出模式。该操作同时设置 `STUDIO_STDOUT_MODE`、`HAS_STUDIO_WEB_SOCKET` 和 `STUDIO_WEB_SOCKET_CONNECTED` 三个标志。启用后应用通过 stdout 以 JSON Lines 格式与 Studio 通信。典型用途：嵌套在 shell 管道中的无头测试或 CI 集成。

### `set_control_channel(rx: Receiver<StudioToApp>)`
注入外部的 `StudioToApp` 消息接收通道。事件循环定期从此通道收消息并分发。发送方应在投递消息后调用 `SignalToUI::set_ui_signal()` 唤醒事件循环。用于外部进程（如 Studio remote bridge）注入控制指令。

### `local_profile_capture_enabled() / set_local_profile_capture_enabled()`
管理本地性能分析开关。禁用时清空累积的样本缓冲区，避免残留数据。

### `take_local_profile_samples() -> Vec<LocalProfileSample>`
原子地取出并清空样本缓冲区。使用 `drain(..)` 避免内存拷贝。每次事件循环迭代结束时由性能分析模块调用此方法将样本发送到 Studio。

### `capture_local_profile_sample(msg: &AppToStudio)`
根据消息类型捕获性能分析样本：
- `EventSample` 且事件名为 "Draw" → 记录为 `LocalProfileSample::Event`
- `GPUSample` → 记录为 `LocalProfileSample::GPU`
- `GCSample` → 记录为 `LocalProfileSample::GC`
- 其他消息类型忽略

样本超过 `LOCAL_PROFILE_SAMPLE_BUFFER_LIMIT` (16,384) 时，丢弃最早的样本（从头部 drain）。最后调用 `SignalToUI::set_ui_signal()` 通知 UI 线程有新的样本数据待处理。

### `run_studio_websocket_thread()`
启动后台 WebSocket 消息聚合线程。该线程的核心逻辑：
1. 创建通道，将 `sender` 存储到 `STUDIO_WEB_SOCKET_THREAD_SENDER`。
2. 使用 `spawn_thread` 生成新线程。
3. 在线程中维护 `AppToStudioVec` 累积缓冲区。
4. 以"收集时间窗口"策略批量发送：默认收集 16ms（约 1 帧），但遇到 `BeforeStartup`、`AfterStartup`、`RequestAnimationFrame`、`DrawCompleteAndFlip` 等急迫消息时将窗口缩短为 1ms。
5. 收集期满后，序列化并调用 `studio_ws_send_binary` 发送，清空缓冲区。
6. 循环直到收到 `Terminate` 或连接中断。退出前将 sender 槽位置为 `None`。

这种**批量发送设计**避免了每条消息都产生一次 WebSocket 帧的开销，大幅降低了高频帧消息（如缩略图更新）的流量。

### `start_studio_websocket(studio_http: &str)`
建立 Studio WebSocket 连接：
1. 空 URL 时直接返回（禁用状态）。
2. 保存 URL 到 `self.studio_http`。
3. 在非 tvOS/iOS 平台上设置标志位，构造 HTTP Upgrade 请求，设置 `WebSocketTransport::PlainTcp`。
4. 保存 `NetworkRuntime` 引用。
5. 调用 `self.net.ws_open` 发起 WebSocket 连接。
6. 失败时重置所有标志位并清理 runtime 引用。

### `stop_studio_websocket()`
停止连接和线程：
1. 调用 `ws_close` 关闭 WebSocket。
2. 清空 runtime 引用。
3. 重置所有标志位。
4. 通过 sender 发送 `Terminate` 消息。

### `start_studio_websocket_delayed()`（仅 tvOS/iOS）
与 `start_studio_websocket` 功能相同但不需要 URL 参数（使用已存储的 `self.studio_http`）。Apple TV 和 iOS 平台上由于网络栈初始化时机差异，可能存在延迟启动需求。

### `init_websockets(studio_http: &str)`
初始化入口：先启动后台线程，再发起 WebSocket 连接。两步顺序执行，线程必须在连接建立之前运行。

### `recv_studio_websocket_message() -> Option<WebSocketMessage>`
事件循环中收帧的核心方法。在一个循环中调用 `self.net.recv()` 获取 `NetworkResponse`：
- 处理 Studio 套接字的 `WsOpened`/`WsClosed`/`WsError`，更新标志并返回纯 WebSocket 生命周期事件。
- 处理 Studio 套接字的 `WsMessage`，返回原始的 `Binary` 或 `String` 数据。
- 其他套接字的消息：投递给 `handle_script_web_socket_event`（JS 脚本 WebSocket）、`handle_script_network_events`（脚本网络事件）和 `Event::NetworkResponses`（应用层网络事件）。

### `send_studio_message(msg: AppToStudio)`
发送 Studio 消息的主入口：
1. 调用 `capture_local_profile_sample` 采样（如果启用）。
2. stdout 模式下直接 JSON 序列化写入 stdout，添加换行，立即 flush。
3. 无 WebSocket 时直接返回（静默丢弃消息）。
4. 优先使用线程的 `sender` 投递（批量模式）。若无线程 sender（如线程尚未启动或已退出），直接通过 `studio_ws_send_binary` 发送。

---

## `WebSocket` 类型别名

```rust
pub type WebSocket = u64;
```

`WebSocket` 只是一个 `u64`，用作连接句柄。Makepad 框架的网络层使用整数 ID（`LiveId` 包装）标识套接字连接，而非胖操作柄。

---

## 设计要点

1. **主线程 + 后台线程双通道**: 后台线程负责消息的序列化和发送批量聚合；主线程负责从网络 runtime 接收消息并进行协议分发。两者通过通道解耦，避免了锁竞争。
2. **批量发送聚合**: 消息被暂存到 `AppToStudioVec`，在收集窗口（16ms 常规 / 1ms 急迫）期满后一次性发送。显著减少了小包高频发送的开销，同时急迫消息的快速通道保证了响应性。
3. **双重序列化支持**: 协议同时支持 Binary（`serde_bin`，高性能）和 Text（JSON，可调试）两种格式。Binary 是主路径；Text 在 stdout 模式和调试场景中使用。
4. **HubToClient 防护**: 应用层的 WebSocket 端口只应接收 `StudioToApp` 消息。代码层对 `HubToClient` 有明确防护：仅 `Hello` 被静默接受，其他变体都会被记录警告并丢弃。
5. **本地性能分析集成**: 样本收集、缓冲、过期丢弃、UI 信号通知等机制深度集成在消息发送路径中，减少了额外的线程同步开销。
6. **三态连接管理**: `HAS_STUDIO_WEB_SOCKET`（协议栈启用）、`STUDIO_WEB_SOCKET_CONNECTED`（连接已建立）、`STUDIO_STDOUT_MODE`（stdout 替代模式）三个独立布尔标志提供了精细的状态控制。
