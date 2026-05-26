# web_socket_parser.rs — WebSocket 帧解析器

**File path**: `platform/network/src/web_socket_parser.rs` (384 行)
**Core purpose**: 实现 RFC 6455 WebSocket 帧的解析与构建，包括掩码处理、分片消息识别。

## 枚举

### State (内部)
- `Opcode`, `Len1`, `Len2`, `Len8`, `Data`, `Mask` — 解析器状态机

### WebSocketMessage<'a>
- `Ping(&[u8])`, `Pong(&[u8])`, `Text(&str)`, `Binary(&[u8])`, `Close`

### WebSocketError<'a>
- `OpcodeNotSupported(u8)` — 未知操作码
- `TextNotUTF8(&[u8])` — 非 UTF-8 文本帧

### WebSocketMessageFormat
- `Binary`, `Text` — 帧格式类型

## 常量

- `SERVER_WEB_SOCKET_PING_MESSAGE: [u8; 2]` = `[0x89, 0x00]` (FIN + Ping, 空负载)
- `SERVER_WEB_SOCKET_PONG_MESSAGE: [u8; 2]` = `[0x8A, 0x00]` (FIN + Pong, 空负载)

## 结构体

### WebSocketParser
- 状态机字段: `head[8]`, `head_expected`, `head_written`, `data: Vec<u8>`, `data_len`, `input_read`, `mask_counter`
- 标志位: `is_ping`, `is_pong`, `is_partial`, `is_text`, `is_masked`

#### parse<F>(input, result) — 核心解析方法
- 状态机遍历输入字节，逐步解析帧头和数据
- Opcode → Len1 → [Len2|Len8] → [Mask] → Data
- 每完成一个消息调用 `result(Ok(...))` 闭包
- 支持掩码解扰 (`data[i] ^ mask[counter & 3]`)
- 支持 Ping/Pong/Text/Binary/Close 帧

#### message_to_frame(msg) -> Vec<u8>
- 将 `WebSocketMessage` 编码为完整帧字节（服务器到客户端，无掩码）

#### create_upgrade_response(key) -> String
- 使用 SHA1 + Base64 生成 WebSocket 升级响应头字符串
- 公式: `base64(sha1(key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))`

### WebSocketMessageHeader
- `format`, `len`, `masked`, `data: [u8; 14]`（最大 14 字节帧头）

#### from_len(len, format, masked) -> Self
- 根据负载长度生成帧头：
  - < 126: 2 字节头
  - < 65536: 4 字节头 (16 位长度)
  - 其他: 10 字节头 (64 位长度)
- 掩码模式额外加 4 字节掩码密钥

#### as_slice() -> &[u8]
- 返回帧头字节切片

#### mask() -> Option<&[u8]>
- 返回掩码密钥位置（如果有）

#### random_byte() — 使用 `subsec_nanos()` 生成随机字节

## 类型别名
- `ServerWebSocketMessage`, `ServerWebSocketError` — 服务器端别名
