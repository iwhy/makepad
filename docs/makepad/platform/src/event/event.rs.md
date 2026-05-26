# `event.rs` — 核心事件系统

## 概述

该文件是 Makepad 事件系统的**核心枢纽**，定义了应用生命周期中所有可能的事件类型（`Event` 枚举），以及若干辅助类型（`Hit`、`DragHit`、`Ease`、`Timer`、`NextFrame` 等）。`Event` 是整个框架消息传递的通用载体，涵盖窗口事件、输入事件、生命周期事件、多媒体事件等所有类别。

---

## `Event` 枚举

### 设计原则

`Event` 是一个 `#[derive(Debug)]` 的大枚举。每个变体要么是空元组（如 `Startup`），要么携带特定的事件数据结构（如 `Draw(DrawEvent)`）。枚举的设计遵循以下原则：

1. **平台无关性**：抹平桌面、移动端、Web 等平台的事件差异，向上层提供统一的抽象
2. **可扩展性**：新增事件只需添加新的枚举变体，不影响现有事件处理逻辑
3. **优先级区分**：某些事件（`MouseDown/Up/Move`）建议用户不要直接匹配，而应使用 `Event::hits()` 系列方法获取 `Hit` 枚举

### 生命周期事件

#### `Event::Startup`（编号 1）
- 应用刚刚创建时发送，且仅发送一次
- 对应 Android 的 `onCreate` 回调
- 适合执行一次性初始化、资源加载、任务创建等操作

#### `Event::Shutdown`（编号 2）
- 应用即将被销毁时发送
- 对应 Android 的 `onDestroy` 回调
- **注意**：某些移动平台可能不发送此事件，不应依赖它做关键持久化

#### `Event::QuitRequested(QuitRequestedEvent)`（编号 67）
- 应用收到退出请求时发送，早于 `Shutdown`
- 触发场景：应用菜单退出、Ctrl+C、SIGTERM 等
- 处理逻辑：设置 `handled = true` 可以推迟或取消退出（如用户确认对话框）
- `QuitReason` 枚举区分退出原因：`App`（程序自请求）、`Menu`（菜单操作）、`Signal`（系统信号）

#### `Event::Foreground`（编号 3）
- 应用进入前台且可见时发送
- 可能多次发送：启动后、从后台恢复后
- 对应 Android 的 `onStart`

#### `Event::Background`（编号 4）
- 应用被隐藏至后台时发送
- 适合暂停 UI/动画更新
- 对应 Android 的 `onStop`

#### `Event::Resume`（编号 5）
- 应用在前台且正接收用户输入时发送
- 对应 Android 的 `onResume`

#### `Event::Pause`（编号 6）
- 应用被暂停但仍在前台可见时发送
- 适合保存临时状态
- 对应 Android 的 `onPause`

### 窗口事件

#### `Event::Draw(DrawEvent)`（编号 7）
- 重绘请求事件。`DrawEvent` 结构体包含：
  - `draw_lists: Vec<DrawListId>` — 需要重绘的绘制列表
  - `draw_lists_and_children: Vec<DrawListId>` — 需要重绘的列表及其子列表
  - `redraw_all: bool` — 是否强制全部重绘
  - `xr_state: Option<Rc<XrState>>` — XR 状态（可选）
  - `time: f64` — 事件时间
- **`DrawEvent::will_redraw()`**：判断是否有任何绘制列表需要重绘（`redraw_all` 或任一列表非空则返回 `true`）
- **`DrawEvent::draw_list_will_redraw(cx, draw_list_id)`**：判断指定 `draw_list_id` 是否命中重绘列表。内部实现遍历 `draw_lists`，沿 `codeflow_parent_id` 链向上查找；然后遍历 `draw_lists_and_children`，从 `draw_list_id` 向上查找

#### `Event::WindowGotFocus(WindowId)`（编号 9）
- 窗口获得焦点，成为活动窗口

#### `Event::WindowLostFocus(WindowId)`（编号 10）
- 窗口失去焦点

#### `Event::WindowDragQuery(WindowDragQueryEvent)`（编号 14）
- 平台询问窗口的某点是否可拖动。由 `window.rs` 中的 `WindowDragQueryEvent` 定义

#### `Event::WindowCloseRequested(WindowCloseRequestedEvent)`（编号 15）
- 窗口被请求关闭

#### `Event::WindowClosed(WindowClosedEvent)`（编号 16）
- 窗口已关闭

#### `Event::WindowGeomChange(WindowGeomChangeEvent)`（编号 17）
- 窗口几何信息变更（位置、大小、DPI、安全区等）

#### `Event::VirtualKeyboard(VirtualKeyboardEvent)`（编号 18）
- 虚拟键盘事件。`VirtualKeyboardEvent` 枚举有四个变体：
  - `WillShow{time, height, duration, ease}` — 键盘将显示，含动画时长和缓动函数
  - `WillHide{time, height, duration, ease}` — 键盘将隐藏
  - `DidShow{time, height}` — 键盘已经显示
  - `DidHide{time}` — 键盘已经隐藏
- `height` 字段表示键盘底部遮挡高度（Makepad 布局点）

#### `Event::ClearAtlasses`（编号 19）
- 清除图集缓存

#### `Event::PopupDismissed(PopupDismissedEvent)`（编号 61）
- 弹窗被关闭通知

### 鼠标/触摸/滚动事件

以下事件为"原始"输入事件，框架建议通过 `Event::hits()` 系列方法获取 `Hit` 枚举，而非直接匹配：

#### `Event::MouseDown(MouseDownEvent)`（编号 20）
- 鼠标按键按下

#### `Event::MouseMove(MouseMoveEvent)`（编号 21）
- 鼠标移动

#### `Event::TweakRay(TweakRayEvent)`（编号 59）
- Studio 调试模式的射线检测，不改变正常输入/悬停状态

#### `Event::MouseUp(MouseUpEvent)`（编号 22）
- 鼠标按键释放

#### `Event::MouseLeave(MouseLeaveEvent)`（编号 51）
- 鼠标移出窗口

#### `Event::TouchUpdate(TouchUpdateEvent)`（编号 23）
- 触摸更新（开始、移动、停止等）

#### `Event::LongPress(LongPressEvent)`（编号 24）
- 长按事件

#### `Event::Scroll(ScrollEvent)`（编号 25）
- 滚动事件（鼠标滚轮/触摸滑动）

### 键盘事件

#### `Event::KeyFocus(KeyFocusEvent)`（编号 30）
- 键盘焦点进入某区域

#### `Event::KeyFocusLost(KeyFocusEvent)`（编号 31）
- 键盘焦点离开

#### `Event::KeyDown(KeyEvent)`（编号 32）
- 按键按下

#### `Event::KeyUp(KeyEvent)`（编号 33）
- 按键释放

#### `Event::TextInput(TextInputEvent)`（编号 34）
- 文本输入

#### `Event::TextRangeReplace(TextRangeReplaceEvent)`（编号 35）
- 文本范围替换

#### `Event::TextCopy(TextClipboardEvent)`（编号 36）
- 复制到剪贴板

#### `Event::TextCut(TextClipboardEvent)`（编号 37）
- 剪切到剪贴板

#### `Event::ImeAction(ImeActionEvent)`（编号 58）
- IME（输入法）动作

#### `Event::SelectionHandleDrag(SelectionHandleDragEvent)`（编号 62）
- 移动端选择手柄拖动事件。包含 `SelectionHandleKind`（Start/End）和 `SelectionHandlePhase`（Begin/Move/End）

### 其他事件

#### `Event::LiveEdit`（编号 8）
- 热重载事件（来自 Makepad 实时编译器）

#### `Event::ScriptReapply`（编号 66）
- 脚本重新应用事件。当脚本对象在运行时被修改后（如 `script_eval!`），此事件通知所有 widget 重新应用新值。与 `LiveEdit` 不同，它保留堆对象的值，使用 `Apply::ScriptReapply` 而非 `Apply::Reload`

#### `Event::GameInputConnected(GameInputConnectedEvent)`（编号 11）
- 游戏输入设备连接/断开

#### `Event::NextFrame(NextFrameEvent)`（编号 12）
- 下一帧请求。`NextFrameEvent` 包含 `frame: u64`（帧号）、`time: f64`、`set: HashSet<NextFrame>`（请求集合）

#### `Event::XrUpdate(XrUpdateEvent)`（编号 13）
- XR 状态更新

#### `Event::XrLocal(XrLocalEvent)`（编号 57）
- XR 本地空间事件，包含手指尖端位置等信息

#### `Event::Timer(TimerEvent)`（编号 26）
- 定时器触发。`TimerEvent` 包含可选的 `time` 和 `timer_id`

#### `Event::Signal`（编号 27）
- 信号事件

#### `Event::Trigger(TriggerEvent)`（编号 28）
- 触发事件，包含 `triggers: HashMap<Area, Vec<Trigger>>`。每个 `Trigger` 包含一个 `LiveId` 和来源 `Area`

#### `Event::MacosMenuCommand(LiveId)`（编号 29）
- macOS 菜单命令

#### `Event::Drag(DragEvent)`（编号 38）
- 拖拽进行中

#### `Event::Drop(DropEvent)`（编号 39）
- 拖放释放

#### `Event::DragEnd`（编号 40）
- 拖拽结束

#### `Event::Custom(String)`（编号 60）
- 应用自定义事件，通过 `Cx::send_studio_message(AppToStudio::Custom(..))` 发送

#### `Event::Actions(ActionsBuf)`（编号 52）
- UI 动作缓冲区

#### `Event::AudioDevices(AudioDevicesEvent)`（编号 41）
- 音频设备变更

#### `Event::MidiPorts(MidiPortsEvent)`（编号 42）
- MIDI 端口变更

#### `Event::VideoInputs(VideoInputsEvent)`（编号 43）
- 视频输入设备变更

#### `Event::NetworkResponses(NetworkResponsesEvent)`（编号 44）
- 网络响应

#### `Event::VideoPlaybackPrepared(VideoPlaybackPreparedEvent)`（编号 45）
- 视频播放已准备好

#### `Event::VideoTextureUpdated(VideoTextureUpdatedEvent)`（编号 46）
- 视频纹理已更新

#### `Event::VideoPlaybackCompleted(VideoPlaybackCompletedEvent)`（编号 47）
- 视频播放完成

#### `Event::VideoDecodingError(VideoDecodingErrorEvent)`（编号 48）
- 视频解码错误

#### `Event::VideoPlaybackResourcesReleased(VideoPlaybackResourcesReleasedEvent)`（编号 49）
- 视频播放资源已释放

#### `Event::TextureHandleReady(TextureHandleReadyEvent)`（编号 50）
- 纹理句柄就绪

#### `Event::VideoYuvTexturesReady(VideoYuvTexturesReady)`（编号 65）
- YUV 纹理已分配

#### `Event::VideoSeekableRanges(VideoSeekableRangesEvent)`（编号 63）
- 可 seek 的时间范围

#### `Event::VideoBufferedRanges(VideoBufferedRangesEvent)`（编号 64）
- 已缓冲的时间范围

#### `Event::BackPressed { handled: Cell<bool> }`（编号 53）
- "返回"导航按钮/手势。推荐使用 `Event::back_pressed()` 方法处理而非直接匹配

#### `Event::PermissionResult(PermissionResult)`（编号 54）
- 权限检查/请求结果

#### `Event::ToWasmMsg(ToWasmMsgEvent)`（仅 wasm32）
- WebAssembly 消息传递。包含 `id: LiveId`、`msg: ToWasmMsg`、`offset: usize`

---

## `Event` 方法实现

### `Event::name(&self) -> &'static str`
返回事件的字符串名称。内部调用 `Self::name_from_u32(self.to_u32())`。

### `Event::name_from_u32(v: u32) -> &'static str`
将 `u32` 编号映射为事件名称字符串。使用大 `match` 语句实现双向映射。若编号不匹配任何已知事件，则 `panic!`。

### `Event::to_u32(&self) -> u32`
将事件变体映射为唯一的 `u32` 编号。与 `name_from_u32` 构成完整的双向映射。

### `Event::back_pressed(&self) -> bool`
处理 `BackPressed` 事件的便捷方法：
1. 先判断事件是否是 `BackPressed` 变体
2. 检查 `handled` 是否已被处理（`Cell<bool>`）
3. 若未被处理，标记为 `handled = true` 并返回 `true`
4. 若已处理或不是 `BackPressed`，返回 `false`
5. 这种"一次性消费"模式确保单个返回操作不会被多个 widget 重复处理

### `Event::requires_visibility(&self) -> bool`
判断事件是否需要窗口可见。返回 `true` 的事件：`MouseDown`、`MouseMove`、`TweakRay`、`TouchUpdate`、`Scroll`。这些事件只在窗口可见时才需要处理。

---

## `Hit` 枚举

`Hit` 是 `Event::hits()` 系列方法返回的"命中结果"类型。它将原始输入事件转换为更高级的、已关联到特定区域的交互事件：

- **`KeyFocus(KeyFocusEvent)`** / `KeyFocusLost(KeyFocusEvent)` — 键盘焦点相关
- **`KeyDown(KeyEvent)`** / `KeyUp(KeyEvent)` — 按键事件
- **`Trigger(TriggerHitEvent)`** — 触发器事件
- **`TextInput(TextInputEvent)`** / `TextRangeReplace(TextRangeReplaceEvent)` — 文本事件
- **`TextCopy(TextClipboardEvent)`** / `TextCut(TextClipboardEvent)` — 剪贴板事件
- **`ImeAction(ImeActionEvent)`** — IME 动作
- **`FingerScroll(FingerScrollEvent)`** — 手指滚动
- **`FingerDown(FingerDownEvent)`** — 手指按下
- **`FingerMove(FingerMoveEvent)`** — 手指移动
- **`FingerHoverIn(FingerHoverEvent)`** — 悬停进入
- **`FingerHoverOver(FingerHoverEvent)`** — 悬停中
- **`FingerHoverOut(FingerHoverEvent)`** — 悬停离开
- **`FingerUp(FingerUpEvent)`** — 手指抬起
- **`FingerLongPress(FingerLongPressEvent)`** — 手指长按
- **`SelectionHandleDrag(SelectionHandleDragEvent)`** — 选择手柄拖动
- **`Nothing`** — 无命中

设计意义：`Hit` 对底层的 `MouseDown/Move/Up`、`TouchUpdate`、`Scroll` 等原始事件做了**统一抽象**，让上层 widget 无需关心输入来源是鼠标还是触摸。

---

## `DragHit` 枚举

`DragHit` 是拖放操作的命中结果，由 `Event::drag_hits()` 系列方法返回：

- **`Drag(DragHitEvent)`** — 拖拽进行中
- **`Drop(DropHitEvent)`** — 拖放释放
- **`DragEnd`** — 拖拽结束
- **`NoHit`** — 无命中

---

## `DrawEvent` 结构体

```rust
pub struct DrawEvent {
    pub draw_lists: Vec<DrawListId>,
    pub draw_lists_and_children: Vec<DrawListId>,
    pub redraw_all: bool,
    pub xr_state: Option<Rc<XrState>>,
    pub time: f64,
}
```

### `DrawEvent::will_redraw()`
简单判断：若 `redraw_all == true` 或任一列表非空，则返回 `true`。

### `DrawEvent::draw_list_will_redraw(cx, draw_list_id)`
判断指定的 `draw_list_id` 是否需要重绘。实现分为两步：
1. **检查 `draw_lists`**：遍历每个 `check_draw_list_id`，沿 `codeflow_parent_id` 链向上查找（从子到父），若 `draw_list_id` 出现在某条链中，则需要重绘
2. **检查 `draw_lists_and_children`**：从 `draw_list_id` 开始沿 `codeflow_parent_id` 链向上查找（从子到父），若某条链的父节点命中 `draw_lists_and_children`，则需要重绘

---

## `Ease` 枚举 — 缓动函数系统

`Ease` 枚举定义了大量缓动（缓动函数），支持 `Script` 和 `ScriptHook`，可直接在 DSL 中使用。

### 支持的缓动类型

| 类别 | 变体 | 描述 |
|------|------|------|
| 线性 | `Linear` | `f(t) = t` |
| 常量 | `None` | 始终返回 1.0 |
| 常量 | `Constant(f64)` | 始终返回给定常量的 clamp 值 |
| 二次 | `InQuad` / `OutQuad` / `InOutQuad` | `t²` 风格 |
| 三次 | `InCubic` / `OutCubic` / `InOutCubic` | `t³` 风格 |
| 四次 | `InQuart` / `OutQuart` / `InOutQuart` | `t⁴` 风格 |
| 五次 | `InQuint` / `OutQuint` / `InOutQuint` | `t⁵` 风格 |
| 正弦 | `InSine` / `OutSine` / `InOutSine` | 基于 `sin()` |
| 指数 | `InExp` / `OutExp` / `InOutExp` | 基于 `2ˣ` |
| 圆形 | `InCirc` / `OutCirc` / `InOutCirc` | 基于 `√(1 - t²)` |
| 弹性 | `InElastic` / `OutElastic` / `InOutElastic` | 弹性弹跳效果 |
| 回退 | `InBack` / `OutBack` / `InOutBack` | 先反向再正向 |
| 弹跳 | `InBounce` / `OutBounce` / `InOutBounce` | 球落地弹跳效果 |
| 指数衰减 | `ExpDecay { d1, d2, max }` | 自定义指数衰减 |
| 幂函数 | `Pow { begin, end }` | 自定义幂曲线 |
| 贝塞尔 | `Bezier { cp0, cp1, cp2, cp3 }` | 三次贝塞尔曲线 |

### `Ease::map(t: f64) -> f64`

核心方法，对输入 `t`（通常为 0.0~1.0）应用缓动变换。

**`ExpDecay` 的实现逻辑**：
1. 若 `t > 0.999`，直接返回 1.0（避免浮点计算误差）
2. 用迭代法计算衰减步数：从 `dt = 1.0` 开始，每步乘以 `di`，同时 `di` 自身乘以 `d2`（双重衰减），直到 `dt < 0.001` 或达到最大步数
3. 然后根据 `t` 线性定位到某两步之间进行插值（lerp），返回 `1.0 - dt_interpolated`

**`Pow { begin, end }` 的实现逻辑**：
1. 对 `begin` 和 `end` 分别 clamp 至 ≥ 1.0（防止除以零）
2. 计算参数 `a` 和 `b`，构造有理函数曲线
3. 使用 `(-a * b + b * a * t2) / (a * t2 - b)` 形式，其中 `t2 = t^t`（幂的幂）以形成非线性映射

**`Bezier` 的实现逻辑**（三次贝塞尔曲线求值）：
1. 快速路径：若 `cp0 ≈ cp1` 且 `cp2 ≈ cp3`，则近似为线性，直接返回 `t`
2. 计算三次贝塞尔函数的系数 `ax, bx, cx, ay, by, cy`
3. **牛顿-拉弗森迭代**（最多 6 次）：沿 X 轴求解参数 `u`，使 `B_x(u) = t`，误差容限为 `t / 200`
4. 若牛顿法收敛失败（导数过小），退化为**二分法**（最多 8 次）
5. 最终用求解出的 `u` 计算 `B_y(u)` 作为返回值
6. 这种方法比代数解法更通用且数值稳定

---

## `VirtualKeyboardEvent` 枚举

移动端虚拟键盘事件模型：
- **`WillShow`**：键盘即将显示，含 `height`（遮挡高度）、`duration`（动画时长）、`ease`（缓动函数）
- **`WillHide`**：键盘即将隐藏
- **`DidShow`**：键盘已显示
- **`DidHide`**：键盘已隐藏

---

## `NextFrameEvent` 和 `NextFrame`

请求下一帧的机制。`NextFrame(pub u64)` 是一个轻量句柄：
- **`NextFrame::is_event(&self, event) -> Option<NextFrameEvent>`**：检查事件是否为 `NextFrame` 且 `set` 中包含自身句柄
- `NextFrameEvent` 中的 `set: HashSet<NextFrame>` 允许在一次帧事件中为多个请求者提供服务

---

## `Timer` 和 `TimerEvent`

定时器机制：

- **`Timer(pub u64)`**：定时器句柄
- **`Timer::is_event(&self, event) -> Option<TimerEvent>`**：检查事件是否为匹配的定时器事件
- **`Timer::is_timer(&self, event) -> Option<TimerEvent>`**：检查 `TimerEvent` 是否匹配
- **`Timer::empty()`**：创建空定时器（编号 0）
- **`Timer::is_empty()`**：判断是否为空定时器

---

## `SelectionHandleDragEvent`

移动端文本选择手柄拖动事件：
- `handle: SelectionHandleKind` — 选择哪个手柄（Start = 起始端, End = 结束端）
- `phase: SelectionHandlePhase` — 拖动阶段（Begin / Move / End）
- `abs: Vec2d` — 绝对位置
- `time: f64` — 时间戳

---

## `Trigger` 和 `TriggerHitEvent`

触发器系统用于事件广播：
- `Trigger { id: LiveId, from: Area }` — 单个触发器，包含 ID 和来源区域
- `TriggerHitEvent(Vec<Trigger>)` — 触发命中事件，包装一组触发器

---

## 类型别名和辅助结构

- `WebSocketErrorEvent { socket_id, error }` — WebSocket 错误通知
- `WebSocketMessageEvent { socket_id, data }` — WebSocket 消息通知
- `ToWasmMsgEvent`（仅 wasm32）— WASM 桥接消息

---

## `QuitRequestedEvent` 和 `QuitReason`

```rust
pub struct QuitRequestedEvent {
    pub reason: QuitReason,
    pub handled: Cell<bool>,
}
```

- `QuitReason`：`App`（程序自请求）、`Menu`（菜单退出）、`Signal`（信号请求）
- `QuitRequestedEvent::new(reason)` — 创建退出请求事件，`handled` 初始化为 `false`
- `QuitRequestedEvent::handle()` — 设置 `handled = true`，阻止退出继续
