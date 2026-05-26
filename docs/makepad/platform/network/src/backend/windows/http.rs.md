# windows/http.rs — Windows WinRT HTTP 客户端

**File path**: `platform/network/src/backend/windows/http.rs` (157 行)
**Core purpose**: 使用 Windows `HttpClient` WinRT API 实现 HTTP 请求，支持流式和非流式读取。

## WindowsHttpSocket

### open(request_id, request, response_sender)
在新线程中执行 async 操作，使用 `makepad_futures_legacy::executor::block_on` 同步等待。

### 内部 async 函数

#### create_request(request) -> windows::core::Result<HttpRequestMessage>
- 创建 `Uri` + `HttpRequestMessage`
- 设置 Headers（`Content-Type` 特殊处理）
- 如有 body：使用 `InMemoryRandomAccessStream` + `DataWriter` 写入流内容

#### streaming_request(request_id, request, response_sender)
- `HttpCompletionOption::ResponseHeadersRead` — 接收 header 后立即返回
- 循环 `ReadAsync` 读取输入流，每次最多 1MB
- 通过 `IBufferByteAccess` 获取原始字节指针
- 发送 `HttpStreamChunk` + 最终 `HttpStreamComplete`

#### non_streaming_request(request_id, request, response_sender)
- `ReadAsBufferAsync` 一次性读取完整响应
- 发送 `HttpResponse`（注意：`status_code: 0` — header 未解析）
