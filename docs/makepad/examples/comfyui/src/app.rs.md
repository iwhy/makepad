# `examples/comfyui/src/app.rs`

ComfyUI + EMDX 示例的核心文件。展示了 Makepad 完整的前端 + 网络 + 异步工作流能力：通过 LLM 生成图像提示词、提交到 ComfyUI 渲染、下载结果并通过 EMDX 协议推送到电子纸显示器。**这是关于 Makepad script DSL 做全栈应用的极佳范本**。

## 整体架构

```
用户输入 Prompt JSON → LLM (OpenAI API) → 生成图像描述 → ComfyUI (Stable Diffusion) → 下载 PNG → EMDX 推送到电子纸显示屏
```

### 使用的 net 能力

- `net.http_request` — HTTP 客户端（LLM API / ComfyUI API）
- `net.http_server` — HTTP 服务器（供 EMDX 拉取内容和图片）
- `net.socket_stream` — TCP Socket（EMDX MDC 协议）
- `net.wake_on_lan` — 唤醒显示器

### 使用的 fs 能力

- `fs.read / fs.write` — 读写本地 prompt.txt 文件

### 使用的 std 能力

- `std.start_timeout / std.stop_timer` — 定时器
- `std.promise / promise.resolve / promise.await` — 异步 Promise
- `std.random_u32` — 随机数

## `script_mod!` 全局变量（行 12-61）

```rust
let self_ip = "10.0.0.112"
let comfy_ip = "10.0.0.165:8000"
let llm_base = "http://10.0.0.217:8080"
let prompt_path = "/Users/admin/prompt.txt"
let rerun_seconds = 20
let comfy_client_id = "8a327a3e4961419ea7386c542f0ea491"
```

这些是硬编码的环境配置，包括：
- `self_ip`: 本机 IP，用于启动 HTTP 服务器供 EMDX 下载
- `comfy_ip`: ComfyUI 服务器地址
- `llm_base`: LLM API 的基础 URL（兼容 OpenAI 接口）
- `prompt_path`: 本地 prompt.txt 文件路径
- `rerun_seconds`: 自动循环间隔（秒）
- `comfy_client_id`: ComfyUI WebSocket 客户端 ID

### Display 和 displays（行 18-24）

```rust
let Display = {mac:"" ip:"" landscape:false prompt:"empty"}.freeze_api()
let displays = [
    Display{mac:"04-E4-B6-F4-5A-8E" ip:"10.0.0.182" landscape:false}
    // ...
]
```

`Display` 结构体定义了电子纸显示器的属性：MAC 地址（用于 Wake-on-LAN）、IP 地址、是否横屏、当前显示提示词。`.freeze_api()` 将结构体变为不可变的 API 模板。

### models 配置（行 26-46）

```rust
let models = {
    flux:{
        file: "./examples/comfyui/flux_dev_full_text_to_image.json"
        sampler: "31"   // ComfyUI 工作流中的节点 ID
        image: "27"     // empty latent image 节点 ID
        prompt: "41"    // CLIP text encode 节点 ID
        save: "9"       // save image 节点 ID
        width: 1600
        height: 896
    }
    qwen:{ ... }  // Qwen 模型配置
}
let model = models.flux
```

`models` 对象包含 Flux 和 Qwen 两种模型的 ComfyUI 工作流配置。每个属性都是工作流 JSON 中的节点 ID，`comfy_render` 函数会动态修改这些节点的输入值。

## UI 组织

### 状态变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `display_iter` | `int` | 轮换显示器的迭代索引 |
| `is_running` | `bool` | 是否有渲染任务正在执行 |
| `auto_enabled` | `bool` | 是否启用自动循环模式 |
| `auto_timer` | `timer` or `nil` | 自动调度的定时器 |
| `messages` | `array` | LLM 对话历史 |
| `current_content_json` | `string` | EMDX content.json 内容 |
| `current_image_data` | `bytes` | 当前下载的图像数据 |

### UI 辅助函数（行 56-108）

| 函数 | 功能 |
|------|------|
| `ui_log(line)` | 向日志文本框追加一行 |
| `set_status(text)` | 更新状态标签并追加日志 |
| `set_progress(text)` | 更新进度标签 |
| `set_display(text)` | 更新当前显示器标签 |
| `set_last_prompt(text)` | 更新最后发送的提示词 |
| `set_preview_path(text)` | 更新预览文件路径 |
| `set_preview_image(bytes)` | 通过 `load_image_from_data_async` 异步加载预览图像 |
| `refresh_auto_button()` | 根据 `auto_enabled` 切换按钮文字 |
| `default_prompt_json()` | 返回默认的 prompt JSON |
| `load_prompt_into_ui()` | 从文件读取 prompt 并加载到编辑器 |
| `save_prompt_from_ui()` | 将编辑器内容保存到文件 |

### 定时控制函数（行 110-140）

**`schedule_auto_rerun()`**: 如果自动模式启用，调度一个 `rerun_seconds` 秒后执行 `post()` 的定时器。

**`start_auto_loop()`**: 启用自动模式，如果不在运行则调度首次重跑。

**`stop_auto_loop()`**: 禁用自动模式，取消挂起的定时器。

**`sleep_seconds(seconds)`**: 用 Promise + Timeout 实现的异步睡眠。

## LLM 集成（行 142-193）

### `llm_stream_delta_content(chunk)`（行 142-153）

解析 Server-Sent Events (SSE) 数据流：
1. 移除 "data: " 前缀
2. 跳过空行或 "[DONE]" 标记
3. 解析 JSON 并提取 `choices[0].delta.content`

### `llm_completion(messages)`（行 155-192）

发送流式 LLM 请求：
1. 创建 Promise
2. 构造 POST 请求到 `llm_base + "/v1/chat/completions"`
3. 设置 `is_streaming: true`，接收 SSE 流
4. `on_stream` 回调实时解析流式数据并更新 UI
5. `on_complete` 时 resolve Promise

## ComfyUI 集成（行 194-282）

### `comfy_image_download(image)`（行 194-215）

通过 HTTP GET 下载 ComfyUI 输出图像。URL 包含 `filename`、`subfolder`、`type` 三个查询参数。

### `comfy_last_image(prompt_id, model)`（行 217-235）

查询 ComfyUI 的 `/history/{prompt_id}` API，从输出的 `model.save` 节点获取图像元数据。

### `comfy_wait_for_image(prompt_id, model, timeout_seconds)`（行 237-247）

轮询 `comfy_last_image`，每秒检查一次。超时返回 `nil`。

### `comfy_render(prompt, display, model)`（行 249-282）

核心渲染函数：
1. 读取 ComfyUI 工作流 JSON 文件
2. 提取 `style_and_keywords` 和 `visual_description` 填入 prompt 节点
3. 设置随机种子
4. 根据显示器横屏/竖屏模式动态调整图像宽高
5. POST 到 ComfyUI `/prompt` API

## HTTP 服务器（行 284-343）

### `http_response(headers, displays)`（行 284-337）

路由处理函数：

| 路径 | 响应 |
|------|------|
| `/content.json` | 返回 EMDX 播放列表 JSON |
| `/image` | 返回当前图像 PNG 数据，无图像时返回 404 |
| `/?{index}` | 返回对应显示器的当前提示词文本 |
| 其他 | 返回 EMDX 的 HTML body (`mod.edmx.http_body`) |

### HTTP 服务器启动（行 339-343）

```rust
let http_server = net.http_server(net.HttpServerOptions{
    listen:"0.0.0.0:3000"
}, net.HttpServerEvents{
    on_get: |headers| http_response(headers, displays)
})
```

监听 3000 端口，处理 `on_get` 回调。

## 核心工作流：`post()` 函数（行 345-477）

这是整个应用的**主流程函数**：

### 步骤分解

1. **检查运行状态**（行 346-351）：如果已经在运行，忽略请求
2. **解析 Prompt JSON**（行 354-361）：从编辑器读取并解析
3. **保存 Prompt 到文件**（行 363）
4. **提取 system 和 user prompt**（行 365-395）
5. **管理对话历史**（行 386-395）：超过 40 条记录时清空；`clear` 标记或首次执行时重置对话
6. **轮换显示器**（行 397-399）：通过 `display_iter` 循环选择
7. **调用 LLM 生成图像提示**（行 401-416）：流式获取结果，尝试解析 JSON
8. **提交到 ComfyUI 渲染**（行 421-437）：提交工作流，等待完成
9. **下载结果图像**（行 445-452）
10. **推送到 EMDX 显示器**（行 454-470）
11. **清理**（行 472-476）：调度自动重跑、重置状态

### 错误处理

每个步骤都有完善的错误处理：出错时重置 `is_running = false`、恢复按钮文字 "Run Now"、显示错误信息。

## 主 UI 布局（行 479-641）

```
Window (980×760)
├── Header: "ComfyUI + EMDX" (圆角深色背景)
├── Content (左右分栏)
│   ├── 左栏 (Flex)
│   │   ├── 操控按钮行 (Load/Save/Run Now/Pause Auto)
│   │   ├── Prompt 编辑器 (多行 TextInput, 140px)
│   │   ├── 状态面板 (Status/Progress/Display)
│   │   ├── Last Prompt 预览 (只读 TextInput, 56px)
│   │   └── 日志滚动区域 (ScrollYView + 只读 TextInput)
│   └── 右栏 (320px 固定宽度)
│       ├── 预览图像标题
│       └── RoundedView 包含 Image + 文件路径标签
```

### 关键 UI 组件

- **ButtonFlat**: Load/Save/Pause Auto 使用扁平按钮
- **Button**: "Run Now" 使用常规按钮（更醒目）
- **TextInput**: 多行 JSON 编辑器、只读日志和提示词预览
- **Image + ImageFit.Smallest**: 等比例预览图像
- **ScrollYView**: 日志区域可滚动

## `App` 结构体（行 643-647）

```rust
#[derive(Script, ScriptHook)]
pub struct App {
    #[live] ui: WidgetRef,
}
```

这个示例中 App 结构体非常简单，因为大部分状态和逻辑都通过 `script_mod!` 的全局变量管理。

## 事件处理

### `MatchEvent::handle_actions`（行 649-651）

App 级别无按钮事件处理，所有交互通过 script 闭包（`on_click: || post()`）直接处理。

### `AppMain::handle_event`（行 653-665）

1. 调用 `match_event` 分派动作
2. 调用 `ui.handle_event` 处理子组件事件
3. `App::script_mod` 按顺序注册：widgets → edmx socket 扩展 → edmx script → 主 UI

## 脚本 UI 的强大之处

这个示例展示了 Makepad script DSL 的独特优势：

1. **完整闭包回调**：按钮事件直接在 DSL 中绑定 `on_click: || post()`，无需 Rust 的 MatchEvent 处理
2. **异步工作流**：通过 `std.promise()` + `.await()` 实现直观的异步代码
3. **HTTP 服务器和客户端**：同语言实现，无需额外框架
4. **内置 Timer/Promise 系统**：自动循环和调度
5. **文件系统访问**：本地持久化 prompt 配置

## 总结

这个示例是 Makepad 脚本系统能力的全面展示：结合 LLM API 调用、ComfyUI 图像生成、EMDX 电子纸推送协议和本地 HTTP 服务器，构建了一个完整的 AI 图像生成与分发系统。
