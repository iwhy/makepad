# utils.rs — HTTP 工具函数与头部解析

**File path**: `platform/network/src/utils.rs` (190 行)
**Core purpose**: 提供 HTTP 服务器和客户端共用的工具函数，包括 TCP 流写入、HTTP 头部解析和 URL 路径处理。

## 公有函数

### write_bytes_to_tcp_stream_no_error(stream, bytes) -> bool
- 将字节完整写入 TCP 流，处理部分写入
- 返回 `true` 表示发生错误

### http_error_out(stream, code)
- 向 TCP 流写入 `HTTP/1.1 {code}\r\n\r\n` 并关闭连接

### split_header_line(inp, what) -> Option<&str>
- 不区分大小写地匹配 HTTP 头部行前缀
- 返回头部值部分（不包括 `\r\n`）

### parse_url_path(url, append_index_html) -> Option<(String, Option<String>)>
- 从 HTTP 请求行解析路径和查询参数
- `append_index_html`: 路径以 `/` 结尾时自动添加 `index.html`

## 结构体

### HttpServerHeaders
- `addr: SocketAddr`, `addr_text: String` — 客户端地址
- `lines: Vec<String>` — 原始头部行
- `verb: String` — HTTP 动词（GET/POST 等）
- `path: String`, `path_no_slash: String`, `search: Option<String>` — URL 解析结果
- `content_length: Option<u64>`, `accept_encoding: Option<String>`
- `sec_websocket_key: Option<String>` — WebSocket 升级检测

### HttpServerHeaders::from_tcp_stream(stream) -> Option<Self>
- 使用 `BufReader` 逐行读取 HTTP 请求头
- 解析第一行获取动词和路径
- 提取 `Content-Length`、`Accept-Encoding`、`Sec-WebSocket-Key`
- 内置 4096 行/行的溢出保护
