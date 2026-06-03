# `examples/comfyui/src/edmx.rs`

三星 EMDX（SMART signage MDC）电子纸显示器的底层通信协议实现。包含 Rust 层面的 MDC 帧编解码、Wake-on-LAN 发送、Socket Stream 方法扩展，以及 script 层面（`script_mod!`）的上传工作流。

## 整体架构

```
┌──────────────────────┐     MDC Protocol (TCP :1515)     ┌──────────────┐
│  ComfyUI App (本机)  │ ──────────────────────────────►  │  EMDX Display │
│  Rust: frame 编解码   │    0xAA 0xFF header + checksum   │              │
│  Script: 上传工作流   │ ◄────────────────────────────── │  电子纸屏幕   │
└──────────────────────┘     Wake-on-LAN (UDP :9)         └──────────────┘
```

## Rust 层实现

### `SocketMdcResponse` 结构体（行 13-23）

```rust
#[derive(Script, ScriptHook)]
struct SocketMdcResponse {
    display_id: f64,
    command_id: f64,
    ack: bool,
    payload: Vec<u8>,
}
```

MDC 响应的解析结果类型，标记为 `Script` 和 `ScriptHook` 使其可从脚本调用。

### `SOCKET_MDC_BUFFERS` 线程本地存储（行 25-27）

```rust
thread_local! {
    static SOCKET_MDC_BUFFERS: RefCell<HashMap<ScriptHandle, Vec<u8>>> = ...;
}
```

使用 `thread_local!` + `RefCell<HashMap>` 维护每个 socket 句柄的接收缓冲区。这是必要的，因为 MDC 协议是帧流式的，需要累积不完整的帧数据。

### MDC 协议帧格式

MDC 协议使用以下二进制帧结构：

```
0xAA | 0xFF | display_id(1) | length(1) | cmd_id(1) | payload(length-2) | checksum(1)
```

- **帧头**: `0xAA 0xFF`
- **display_id**: 目标显示器 ID（1 字节）
- **length**: 数据区总长度（包括 cmd_id 和 payload，1 字节）
- **cmd_id**: 命令 ID（1 字节）
- **payload**: 负载数据（length - 2 字节）
- **checksum**: 0xFF 到 payload 末尾所有字节和取模 256

#### `parse_mdc_response_frame`（行 35-85）

```rust
fn parse_mdc_response_frame(buffer: &mut Vec<u8>) -> Result<Option<(u8, u8, bool, Vec<u8>)>, String>
```

有限状态机解析器：

1. 跳过非 `0xAA` 的数据（同步字节查找）
2. 检查组长度为 2 的帧头 `0xAA 0xFF`
3. 计算完整帧长度 `5 + length`（header 5 + payload length）
4. 计算校验和并验证
5. 提取 `ack_or_nak` 字节：`0x41`(ACK) 或 `0x4E`(NAK)
6. 返回 `(display_id, command_id, is_ack, payload)`

关键设计：所有不符合协议的数据都被丢弃，循环直到找到有效帧或数据不足。

#### `build_mdc_frame`（行 87-102）

构造 MDC 帧，payload 超过 255 字节会返回错误。

#### `parse_mac_to_bytes`（行 104-116）

解析 MAC 地址字符串（支持各种分隔符，如 `-`、`:`）为 6 字节数组。先过滤掉非十六进制字符，再按每 2 个字符一组解析。

#### `script_value_to_bytes`（行 118-146）

将脚本值转换为字节数组，支持：
- **字符串**: 直接转为 UTF-8 字节
- **U8/U16/U32 数组**: 元素逐字节转换
- **F32 数组**: 元素逐字节转换
- **ScriptValue 数组**: 每个值必须是 0-255 的数值

### Socket 方法扩展

#### `register_socket_extensions`（行 148-341）

```rust
pub fn register_socket_extensions(vm: &mut ScriptVm) {
    let net = vm.module(id_lut!(net));
    set_script_value_to_api!(vm, net.SocketMdcResponse);
    let socket_stream_type = vm.handle_type(id_lut!(socket_stream));
    // ...
}
```

向 `socket_stream` 类型添加三个自定义方法：

#### `next_mdc`（行 154-217）

```rust
vm.add_handle_method(socket_stream_type, id_lut!(next_mdc), ...)
```

从 socket 读取并解析一个完整的 MDC 响应帧。循环处理：

| 状态 | 处理 |
|------|------|
| 缓冲区中有完整帧 | 解析后返回 `SocketMdcResponse` |
| 数据不足 | `socket_stream_poll` 阻塞等待更多数据 |
| 连接关闭（有错误） | 清除缓冲区，返回 IO 错误 |
| 连接关闭（无错误） | 清除缓冲区，返回 `nil` |
| 需要暂停 | `socket_stream_pause_current` 暂停当前流 |
| 暂停过多 | 返回 `script_err_limit!` |

#### `send_mdc_command`（行 220-249）

向 socket 发送一个 MDC 命令帧。参数为 `command_id`、`display_id`、`data`。验证参数范围（0-255），构建帧后通过 `socket_stream_send_bytes` 发送。

#### `send_mdc_set_content_download`（行 252-292）

发送专门的 MDC 内容下载命令（命令 ID `0xC7`）。构造的数据部分为：`0x53 0x80 | url_length | url_bytes`。URL 长度限制为 255 字节。

这是 EMDX 协议中告诉显示器从某个 URL 下载内容的指令。

#### `wake_on_lan`（行 294-341）

```rust
vm.add_method(net, id_lut!(wake_on_lan), ...)
```

在 `net` 模块层级添加 Wake-on-LAN 功能（不是方法，而是模块函数）：

1. 解析 MAC 地址为 6 字节
2. 构造魔术包：`0xFF × 6 + MAC × 16`
3. 创建 UDP socket 发送广播（默认目标 `255.255.255.255:9`）

## Script 层实现（行 343-467）

### `mod.edmx` 模块

```rust
script_mod! {
    use mod.std
    use mod.net

    mod.edmx = {
        http_body: "..."
        // 函数定义
    }
}
```

#### `http_body`（行 348-363）

一个自包含的 HTML 页面，包含 JavaScript 轮询逻辑：
- 单击页面 → 全屏显示
- 双击 → 重新加载
- JavaScript 每隔 1 秒 fetch 当前路径（携带查询参数），将服务器返回的文本更新到页面上的 `<b>` 元素

这是 EMDX 电子纸显示器的内容呈现页面。

#### `wait_for_socket_text(socket, pass, fail_1, fail_2)`（行 365-374）

通用的 TCP 文本协议交互函数：
1. 循环接收 `socket.next_string()` 数据块
2. 累加到 `text` 缓冲区
3. 如果匹配 `pass` 字符串，返回 `pass`
4. 如果匹配 `fail_1` 或 `fail_2`，返回对应的失败标记
5. 连接关闭返回空字符串

用于 MDC 认证过程的 TLS 握手和密码验证。

#### `build_content_json(local_ip, local_port, file_id, file_size)`（行 377-400）

构造 EMDX 播放列表 JSON：

```json
{
    "schedule": [{
        "start_date": "1970-01-01",
        "stop_date": "2999-12-31",
        "contents": [{
            "image_url": "http://{ip}:{port}/image",
            "file_id": "{file_id}",
            "duration": 91326,
            "file_size": "{file_size}",
            "file_name": "{file_id}.png"
        }]
    }],
    "name": "node-samsung-emdx",
    "content_type": "ImageContent",
    ...
}
```

这是 EMDX 协议要求的播放列表格式，告诉显示器从哪里下载图像并显示多久。

#### `mdc_wait_for_command(socket, command_id)`（行 402-408）

循环调用 `socket.next_mdc()`，直到收到指定 `command_id` 的 MDC 响应。

#### `upload_image(display, content_url, sleep_seconds)`（行 410-465）

EMDX 上传的核心工作流：

```
upload_image 流程:
1. [可选] Wake-on-LAN 唤醒显示器
2. TCP 连接显示器 1515 端口
3. 等待 TLS 握手问候 "MDCSTART<<TLS>>"
4. 启动 TLS 加密
5. 发送密码 "123456"
6. 等待认证结果 (PASS/FAIL)
7. 发送 set_content_download MDC 命令
8. 等待命令 199 (0xC7) 的响应
9. 关闭连接
10. 返回 {is_ok: true/false}
```

认证失败有三种情况：
- `MDCAUTH<<FAIL:0x01>>`：密码错误
- `MDCAUTH<<FAIL:0x02>>`：设备已锁定
- 没有收到 PASS 标记：未知错误

如果 MDC 响应是 NAK，返回错误信息及 NAK payload 内容。

## 总结

`edmx.rs` 是一个完整的硬件通信协议实现示例，展示了 Makepad 的以下能力：

1. **Rust 层协议编解码**：二进制帧的构造、解析、校验
2. **Script 运行时扩展**：通过 `add_handle_method` 和 `add_method` 扩展脚本能力
3. **Socket Stream 集成**：TCP 连接的建立、TLS 加密、数据收发
4. **UDP Broadcast**：Wake-on-LAN 魔术包发送
5. **脚本 DSL 封装**：将复杂的握手流程封装为简单脚本函数
6. **错误处理**：多级认证失败的分级处理
