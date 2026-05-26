# linux/http.rs — Linux HTTP 客户端实现

**File path**: `platform/network/src/backend/linux/http.rs` (387 行)
**Core purpose**: 基于 OpenSSL socket 流的 Linux HTTP/HTTPS 请求实现，支持流式和非流式响应、分块传输编码。

## LinuxHttpSocket

静态方法：

### open(request_id, request, response_sender)
- 设置取消标志并注册到全局 `cancellation_map`
- 在新线程中执行 `run_http_request`
- 完成后从 cancellation map 移除

### cancel(request_id)
- 在 `cancellation_map` 中设置取消标志为 `true`

## 内部函数

### run_http_request
- 解析 URL，确定是否使用 TLS
- 连接 SocketStream（支持 HTTP/HTTPS）
- 写入 HTTP 请求（`write_request`）
- 读取响应头（`read_response_head`）
- 流式模式：
  - 逐块发送 `HttpStreamChunk`
  - 完成后发送 `HttpStreamComplete`
- 非流式模式：
  - 读取完整 body
  - 如果 `Transfer-Encoding: chunked`，调用 `decode_chunked_body`

### write_request(stream, request, split, use_tls)
- 构造 HTTP 请求行和头部
- 自动计算 Content-Length（如未在 headers 中提供）
- 写入请求行 + headers + body

### read_response_head(stream, cancel_flag) -> (status_code, headers_string, body_prefix, chunked)
- 逐块读取直到发现 `\r\n\r\n`
- 解析状态行获取状态码
- 检查 `Transfer-Encoding: chunked`

### decode_chunked_body(raw) -> Result<Vec<u8>, String>
- 手动解析 HTTP 分块编码
- 遍历 `size\r\ndata\r\n` 格式，size 为十六进制数

### write_all(stream, data)
- 完整写入，处理部分写入和 WouldBlock

### find_crlf / find_header_end
- 查找 `\r\n` 和 `\r\n\r\n` 位置
