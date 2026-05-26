# `headless.rs` — 标准库无头集成测试

## 文件位置
`platform/script/std/tests/headless.rs`

---

## 总体职责
这个文件包含了一个完整的端到端测试，验证 Splash 标准库的 HTTP 请求 + Promise 机制在无头（headless）模式下正确工作。测试模拟了一个简单的 `NetworkBackend`，不涉及真实的网络 I/O，在单线程同步环境中验证 `pump_network_runtime`、`with_vm`、`script_mod` 以及 Promise 调度机制的正确性。

---

## 测试结构和 Setup

### `struct TestBackend`
- 实现了 `NetworkBackend` trait，是 makepad_network 的模拟后端。
- `http_start()` 立即向 `EventSink` 发射一个 `HttpResponse`，状态码 200，body 为 `b"ok"`。`request_id` 使用调用方传入的 ID。
- 其他方法（`http_cancel`、`ws_open`、`ws_send`、`ws_close`）均直接返回 `Ok(())`，不做实际操作。

### 测试初始化
- 创建 `NetworkRuntime::with_backend(Arc::new(TestBackend))`，包装为 `Arc`。
- 创建 `ScriptStd::with_network_runtime(runtime)`，注入网络运行时。
- 创建 `ScriptVmBase::new()`，包装在 `Some(Box::new(...))` 中。
- `host` 设置为空的 `()` 元组。

---

## 测试用例

### `fn headless_http_request_resolves_promise_through_script_std()`
- **Phase 1 — 执行脚本**：在 `with_vm` 中调用 `script_mod(vm)` 注册标准库，然后 `eval` 一段 Splash 脚本：
  - 创建一个 `std.promise()` 得到句柄 `p`。
  - 创建 `HttpRequest{url: "https://example.com", method: net.HttpMethod.GET}`。
  - 调用 `net.http_request(req, net.HttpEvents{...})`，注册 `on_response` 回调（将状态码 resolve 到 promise）和 `on_error` 回调（将 -1 resolve 到 promise）。
  - 返回 promise 句柄。
- **Phase 2 — 验证请求状态**：断言返回的值为句柄类型，并且 `std.data.http_requests.len() == 1`（请求已注册）。
- **Phase 3 — 泵送网络事件**：调用 `pump_network_runtime(...)`，它从 `NetworkRuntime` 中排干 `TestBackend` 立即发射的 `HttpResponse`。
  - 断言返回了 1 个 response，并且处理后 `http_requests` 列表变为空。
- **Phase 4 — 验证 Promise 结果**：获取 `std.data.tasks.tasks` 列表，找到与该 promise handle 匹配的任务。
  - 检查任务队列（`task.queue.as_array()`）的底层存储类型为 `ScriptArrayStorage::ScriptValue`。
  - 断言队列中只有一个值，且该值为 `f64` 类型，值为 `200.0`（即来自 `on_response` 回调的 `res.status_code`）。

**测试结论**：该测试验证了整个流水线——脚本注册 → HTTP 请求发起 → 网络后端返回 → 网络事件泵送 → Promise resolve → 最终结果存储在任务队列中。这是 Splash 标准库事件驱动架构的端到端正确性验证。
