# runtime.rs — 网络运行时封装

**File path**: `platform/network/src/runtime.rs` (168 行)
**Core purpose**: 提供 `NetworkRuntime` 类型作为网络后端的同步 facade，通过 mpsc channel 将异步网络事件桥接到调用线程。

## 结构体

### NetworkConfig
- `backend: Option<Arc<dyn NetworkBackend>>` — 可选的自定义后端

### NetworkRuntime
- `backend: Arc<dyn NetworkBackend>` — 持有的后端实现
- `sink: EventSink` — 事件发送端（可设置 wake_fn）
- `receiver: Mutex<Receiver<NetworkResponse>>` — 事件接收端

## 关键方法

### 构造
- `new(config: NetworkConfig)` — 使用默认后端（或配置覆盖）创建运行时
- `with_backend(backend)` — 直接使用指定后端创建

### 事件循环集成
- `set_wake_fn(wake_fn)` — 设置唤醒函数（例如用于通知 UI 线程处理事件）

### HTTP API
- `http_start(request_id, request)` — 发起 HTTP 请求
- `http_cancel(request_id)` — 取消进行中的请求

### WebSocket API
- `ws_open(socket_id, request)` — 打开 WebSocket
- `ws_send(socket_id, message)` — 发送消息
- `ws_close(socket_id)` — 关闭连接

### HTTP 服务器
- `start_http_server(http_server)` — 启动内嵌 HTTP 服务器（返回线程 JoinHandle）

### 事件接收
- `try_recv()` — 非阻塞读取
- `recv()` — 阻塞读取
- `recv_timeout(duration)` — 超时读取

## 实现细节
- 所有网络操作通过 trait 对象 `dyn NetworkBackend` 动态分发
- 后端通过 `EventSink::emit()` 发送事件，`NetworkRuntime` 在同一 channel 上接收
- wake_fn 在每次 emit 后被调用，实现跨线程通知
