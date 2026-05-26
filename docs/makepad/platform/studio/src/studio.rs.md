# studio.rs — Studio 客户端与协议数据类型

## 概述

`studio.rs` 定义了 Makepad Studio 远程协议的核心数据类型——这是运行中的应用程序与 Studio IDE 之间双向通信的消息结构。该文件包含以下内容：

- **AppToStudio** 枚举 — 应用发给 Studio 的消息
- **StudioToApp** 枚举 — Studio 发给应用的消息
- 辅助请求/响应结构体（Screenshot、RunViewFrame、WidgetTreeDump 等）
- 远程输入事件结构体（RemoteMouseDown、RemoteKeyModifiers 等）
- 编辑器操作结构体（PatchFile、EditFile、JumpToFile 等）
- 性能分析样本结构体（EventSample、GPUSample、GCSample）

所有类型均通过 `makepad_micro_serde` 的 `SerBin/DeBin/SerJson/DeJson` 派生支持二进制和 JSON 序列化。

---

## 性能分析样本

### EventSample
```rust
pub struct EventSample {
    pub event_u32: u32,    // 事件类型标识
    pub event_meta: u64,   // 事件元数据
    pub start: f64,        // 开始时间戳
    pub end: f64,          // 结束时间戳
}
```

### GPUSample
```rust
pub struct GPUSample {
    pub start: f64, pub end: f64,
    pub draw_calls: u64, pub instances: u64, pub vertices: u64,
    pub instance_bytes: u64, pub uniform_bytes: u64,
    pub vertex_buffer_bytes: u64, pub texture_bytes: u64,
}
```

### GCSample
```rust
pub struct GCSample {
    pub start: f64, pub end: f64,
    pub heap_live: u64,  // 堆上活跃字节数
}
```

### LocalProfileSample
```rust
pub enum LocalProfileSample {
    Event(EventSample),
    GPU(GPUSample),
    GC(GCSample),
}
```

---

## Studio 日志与编辑器操作

### StudioLogItem
```rust
pub struct StudioLogItem {
    pub file_name: String,
    pub line_start: u32, pub line_end: u32,
    pub column_start: u32, pub column_end: u32,
    pub message: String,
    pub explanation: Option<String>,
    pub level: LogLevel,
}
```

### JumpToFile
```rust
pub struct JumpToFile {
    pub file_name: String,
    pub line: u32, pub column: u32,
}
```

### PatchFile
支持字符级替换的原子编辑操作。
```rust
pub struct PatchFile {
    pub file_name: String,
    pub line: u32,
    pub column_start: u32, pub column_end: u32,
    pub undo_group: u64,   // 撤销分组 ID
    pub replace: String,
}
```

### EditFile
支持行级替换的批量编辑操作。
```rust
pub struct EditFile {
    pub file_name: String,
    pub line_start: u32, pub line_end: u32,
    pub column_start: u32, pub column_end: u32,
    pub replace: String,
}
```

### SelectInFile
```rust
pub struct SelectInFile {
    pub file_name: String,
    pub line_start: u32, pub line_end: u32,
    pub column_start: u32, pub column_end: u32,
}
```

### SwapSelection
在同一个文件的两个区域或两个文件之间交换选区。
```rust
pub struct SwapSelection {
    pub s1_file_name: String,
    pub s1_line_start: u32, pub s1_line_end: u32,
    pub s1_column_start: u32, pub s1_column_end: u32,
    pub s2_file_name: String,
    pub s2_line_start: u32, pub s2_line_end: u32,
    pub s2_column_start: u32, pub s2_column_end: u32,
}
```

---

## 远程输入事件结构体

这些结构体用于在 Studio 与应用进程之间传输输入事件。

### RemoteKeyModifiers
```rust
pub struct RemoteKeyModifiers {
    pub shift: bool, pub control: bool,
    pub alt: bool, pub logo: bool,
}
```
提供 `into_key_modifiers()` 和 `from_key_modifiers()` 转换到/从 `KeyModifiers`。

### RemoteMouseDown
```rust
pub struct RemoteMouseDown {
    pub button_raw_bits: u32,  // 鼠标按钮位掩码
    pub x: f64, pub y: f64,    // 坐标
    pub time: f64,              // 时间戳
    pub modifiers: RemoteKeyModifiers,
}
```

### RemoteMouseMove
```rust
pub struct RemoteMouseMove {
    pub time: f64,
    pub x: f64, pub y: f64,
    pub modifiers: RemoteKeyModifiers,
}
```

### RemoteTweakRay
```rust
pub struct RemoteTweakRay {
    pub time: f64,
    pub x: f64, pub y: f64,
    pub modifiers: RemoteKeyModifiers,
}
```

### RemoteMouseUp
```rust
pub struct RemoteMouseUp {
    pub time: f64,
    pub button_raw_bits: u32,
    pub x: f64, pub y: f64,
    pub modifiers: RemoteKeyModifiers,
}
```

### RemoteTextInput
```rust
pub struct RemoteTextInput {
    pub time: f64,
    pub window_id: usize,
    pub raw_button: usize,
    pub x: f64, pub y: f64,
}
```

### RemoteScroll
```rust
pub struct RemoteScroll {
    pub time: f64,
    pub sx: f64, pub sy: f64,   // 滚动偏移
    pub x: f64, pub y: f64,      // 鼠标位置
    pub is_mouse: bool,           // 是否来自鼠标滚轮
    pub modifiers: RemoteKeyModifiers,
}
```

---

## AppToStudio 枚举

应用（运行中的程序）发送给 Studio IDE 的消息集合。

```rust
pub enum AppToStudio {
    LogItem(StudioLogItem),           // 日志条目
    EventSample(EventSample),         // 事件性能样本
    GPUSample(GPUSample),             // GPU 性能样本
    GCSample(GCSample),               // GC 性能样本
    JumpToFile(JumpToFile),           // 跳转到文件位置
    SelectInFile(SelectInFile),       // 选中文件区域
    PatchFile(PatchFile),             // 打补丁（字符级编辑）
    EditFile(EditFile),               // 编辑文件（行级替换）
    SwapSelection(SwapSelection),     // 交换选区
    Screenshot(ScreenshotResponse),   // 截图响应
    RunViewFrame(RunViewFrameData),   // RunView 帧数据
    RunViewKeyFocusRect(RunViewKeyFocusRect), // 键盘焦点矩形
    WidgetTreeDump(WidgetTreeDumpResponse), // widget 树转储
    WidgetQuery(WidgetQueryResponse), // widget 查询结果
    WidgetSnapshot(WidgetSnapshotResponse), // widget 快照
    TweakHits(TweakHitsResponse),     // tweak 命中测试结果
    BeforeStartup,                    // 应用启动前回调
    CreateWindow { window_id: usize, kind_id: usize },  // 创建窗口
    AfterStartup,                     // 应用启动后回调
    RequestAnimationFrame,            // 请求动画帧
    SetCursor(MouseCursor),           // 设置鼠标光标
    SetClipboard(String),             // 设置剪贴板内容
    DrawCompleteAndFlip(PresentableDraw), // 渲染完成，交换缓冲区
    Custom(String),                   // 应用自定义事件
}
```

### 包装类型
```rust
pub struct AppToStudioVec(pub Vec<AppToStudio>);
```

---

## StudioToApp 枚举

Studio IDE 发送给运行中应用的消息集合。

```rust
pub enum StudioToApp {
    Screenshot(ScreenshotRequest),               // 请求截图
    RunViewFrameRequest(RunViewFrameRequest),     // 请求 RunView 帧
    WidgetTreeDump(WidgetTreeDumpRequest),        // 请求 widget 树
    WidgetQuery(WidgetQueryRequest),              // 请求 widget 查询
    WidgetSnapshot(WidgetSnapshotRequest),         // 请求 widget 快照
    KeepAlive,                                     // 保活信号
    LiveChange { file_name: String, content: String },  // 热重载文件变化
    Swapchain(SharedSwapchain),                    // 共享交换链
    WindowGeomChange {                             // 窗口几何变化
        dpi_factor: f64, window_id: usize,
        left: f64, top: f64, width: f64, height: f64,
    },
    Tick,                                          // 心跳 tick
    MouseDown(RemoteMouseDown),                    // 鼠标按下
    MouseUp(RemoteMouseUp),                        // 鼠标释放
    MouseMove(RemoteMouseMove),                    // 鼠标移动
    TweakRay(RemoteTweakRay),                      // Tweak 射线
    KeyDown(KeyEvent),                             // 按键按下
    KeyUp(KeyEvent),                               // 按键释放
    TextInput(TextInputEvent),                     // 文本输入
    TextCopy,                                      // 复制文本
    TextCut,                                       // 剪切文本
    Scroll(RemoteScroll),                          // 滚动
    Custom(String),                                // 应用自定义事件
    None,                                          // 空消息
    Kill,                                          // 关闭应用
}
```

### 包装类型
```rust
pub struct StudioToAppVec(pub Vec<StudioToApp>);
```

---

## 请求/响应结构体

### Screenshot
```rust
pub struct ScreenshotRequest { pub request_id: u64, pub kind_id: u32 }
pub struct ScreenshotResponse {
    pub request_ids: Vec<u64>,
    pub png: Vec<u8>,
    pub width: u32, pub height: u32,
}
```

### RunViewFrame
```rust
pub struct RunViewFrameRequest {
    pub window_id: usize, pub frame_id: u64,
    pub width: u32, pub height: u32, pub dpi_factor: f64,
}
pub struct RunViewFrameData {
    pub window_id: usize, pub frame_id: u64,
    pub width: u32, pub height: u32,
    pub codec: Option<FrameCodec>,  // 帧编码方式（None=原始，Some=压缩）
    pub data: Vec<u8>,
}
pub struct RunViewKeyFocusRect {
    pub x: Option<f64>, pub y: Option<f64>,
    pub width: Option<f64>, pub height: Option<f64>,
}
```

### WidgetTreeDump
```rust
pub struct WidgetTreeDumpRequest { pub request_id: u64 }
pub struct WidgetTreeDumpResponse { pub request_id: u64, pub dump: String }
```

### WidgetQuery
```rust
pub struct WidgetQueryRequest { pub request_id: u64, pub query: String }
pub struct WidgetQueryResponse {
    pub request_id: u64,
    pub query: String,
    pub rects: Vec<String>,   // 查询结果矩形区域
}
```

### WidgetSnapshot
```rust
pub struct WidgetSnapshot {
    pub id: String, pub widget_type: String,
    pub window_id: String, pub window_index: usize,
    pub visible: bool, pub enabled: bool,
    pub x: i64, pub y: i64, pub width: i64, pub height: i64,
    pub text: Option<String>, pub value: Option<String>,
    pub checked: Option<bool>, pub selected: Option<String>,
}
pub struct WidgetSnapshotRequest { pub request_id: u64 }
pub struct WidgetSnapshotResponse {
    pub request_id: u64,
    pub widgets: Vec<WidgetSnapshot>,
}
```

### TweakHitsResponse
```rust
pub struct TweakHitsResponse {
    pub window_id: usize, pub dpi_factor: f64,
    pub ray_x: f64, pub ray_y: f64,
    pub left: f64, pub top: f64, pub width: f64, pub height: f64,
    pub widget_uids: Vec<u64>,
}
```

## 序列化辅助方法

```rust
impl AppToStudio {
    pub fn to_json(&self) -> String   // JSON 序列化 + 换行
}
impl StudioToApp {
    pub fn to_json(&self) -> String   // JSON 序列化 + 换行
}
```
