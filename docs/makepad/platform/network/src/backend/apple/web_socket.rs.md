# apple/web_socket.rs — Apple NSURLSession WebSocket

**File path**: `platform/network/src/backend/apple/web_socket.rs` (187 行)
**Core purpose**: 使用 `NSURLSessionWebSocketTask` API 实现 Apple 平台的原生 WebSocket 客户端。

## 委托类

### web_socket_delegate_class() / define_web_socket_delegate()
- `MakepadNSURLSessionWebSocketDelegate`
- `didOpenWithProtocol` — 记录连接打开
- `didCloseWithCode` — 记录关闭事件

## AppleWebSocket

### 字段
- `data_task: Arc<ObjcId>` — NSURLSessionWebSocketTask 的引用
- `rx_sender: Sender<WebSocketMessage>` — 回调事件发送端

### open(socket_id, request, rx_sender) -> Self
- 构造 NSURLSession（`ignore_ssl_cert` 时使用自定义 delegate 跳过验证）
- 创建 `webSocketTaskWithRequest`
- 设置 `setMaximumMessageSize(5MB)`
- 通过递归 tail-call `set_message_receive_handler` 注册消息接收回调（Block）
- 消息类型 0 → Binary，其他 → String
- 启动 task (`resume`)

### send_message(message) -> Result<(), ()>
- 将消息转换为 `NSURLSessionWebSocketMessage`：
  - `String` → `initWithString:`
  - `Binary` → `initWithData:`
  - `Closed` → `cancel` task
- 通过 `sendMessage:completionHandler:` 异步发送

### close()
- 调用 `cancel` 取消 WebSocket task

## 内部函数

### set_message_receive_handler(data_task_ref, rx_sender)
- 递归注册 `receiveMessageWithCompletionHandler` Block
- 收到消息后转换为 `WebSocketMessage::Binary` 或 `String`
- 每次收到消息后重新注册 handler（Apple 的 API 要求单次注册）
