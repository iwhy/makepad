# apple/http.rs — Apple NSURLSession HTTP 请求实现

**File path**: `platform/network/src/backend/apple/http.rs` (354 行)
**Core purpose**: 通过 Objective-C `NSURLSession` API 实现 HTTP 请求发送和响应处理，支持流式和非流式模式。

## 委托类

### UrlSessionDataDelegateContext
- `sender: Sender<NetworkResponse>`, `request_id: LiveId`, `metadata_id: LiveId`
- 通过 `context_box` ivar 传递给 Objective-C runtime

### url_session_data_delegate_class()
- 懒初始化 `MakepadNSURLSessionDataDelegate` 类
- 回调方法：
  - `didReceiveResponse` — 允许继续（调用 `completionHandler(.Allow)`）
  - `didReceiveData` — 从 `context_box` 恢复上下文，发送 `HttpStreamChunk`
  - `didCompleteWithError` — 发送 `HttpError` 或 `HttpStreamComplete`

### url_session_delegate_class() / define_url_session_delegate()
- `MakepadNSURLSessionDelegate` 类
- `didReceiveChallenge` — 处理 SSL 挑战：如果有 `serverTrust`，用 `NSURLCredential(trust:)` 应答
- 用于 `ignore_ssl_cert` 模式

## 公有函数

### make_ns_request(request: &HttpRequest) -> ObjcId
- 将 `HttpRequest` 转换为 `NSMutableURLRequest`
- 设置 URL、HTTP Method、Headers、Body

## AppleHttpRequests

### 字段
- `requests: Vec<HttpReq>` — 每个请求包括 `request_id` 和 `data_task` (RcObjcId)

### make_http_request(request_id, request, networking_sender)
- 构造 `NSURLSession`（根据 `ignore_ssl_cert` 使用不同委托）
- 流式模式：使用 `dataTaskWithRequest` + `setDelegate` 实现逐块通知
- 非流式模式：使用 `dataTaskWithRequest:completionHandler:` block 一次性获取完整响应
- 响应处理包括解析状态码、header 字段和 body

### cancel_http_request(request_id)
- 调用 `[dataTask cancel]` 并移除请求记录

### handle_response(response)
- 对于 `HttpError` / `HttpResponse` / `HttpStreamComplete` 响应，自动清理请求记录
