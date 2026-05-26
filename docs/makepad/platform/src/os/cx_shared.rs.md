# `cx_shared.rs` — 跨平台共享 Cx 方法

## 文件定位

此文件实现了 `Cx` 类型上一系列与平台无关的方法，涵盖绘制通道调度、Studio 远程协议消息处理、事件分发、热重载等核心功能。所有平台（含 Web）共享此文件，避免了相同逻辑的多份实现。

---

## 一、绘制通道（DrawPass）重绘管理

### `repaint_windows(&mut self)`

```rust
pub(crate) fn repaint_windows(&mut self) {
    for draw_pass_id in self.passes.id_iter() {
        match self.passes[draw_pass_id].parent {
            CxDrawPassParent::Window(_) => {
                self.passes[draw_pass_id].paint_dirty = true;
            }
            _ => (),
        }
    }
}
```

- **实现逻辑**：
  1. 遍历 `self.passes` 中所有已注册的绘制通道。
  2. 对每个父类型为 `Window` 的通道，将其 `paint_dirty` 标志设置为 `true`，标记需要重绘。
  3. 不直接标记非窗口通道——依赖传播由 `compute_pass_repaint_order` 处理。
- **用途**：在需要强制所有窗口重新绘制时调用（例如热重载后、Studio 截屏请求时）。

### `any_passes_dirty(&self) -> bool`

```rust
pub(crate) fn any_passes_dirty(&self) -> bool {
    for draw_pass_id in self.passes.id_iter() {
        if self.passes[draw_pass_id].paint_dirty {
            return true;
        }
    }
    false
}
```

- **实现逻辑**：
  1. 遍历所有绘制通道。
  2. 只要有任何通道的 `paint_dirty == true`，立即返回 `true`。
  3. 全部检查完无脏通道则返回 `false`。
- **用途**：事件循环检查是否需要触发新一帧绘制。

### `compute_pass_repaint_order(&mut self, passes_todo: &mut Vec<DrawPassId>)`

```rust
pub(crate) fn compute_pass_repaint_order(&mut self, passes_todo: &mut Vec<DrawPassId>)
```

- **职责**：计算下一帧需要绘制的通道集合及其拓扑顺序。
- **实现逻辑**（分两阶段）：
  - **阶段一：脏标记传播（loop）**
    1. 循环遍历所有通道，如果某通道 `paint_dirty` 且其父类型为 `DrawPass(parent_pass_id)`，则将父通道也标记为 `paint_dirty = true`。
    2. 重复此过程直到没有新的脏标记产生。这确保了子通道的脏状态向上传播到整条依赖链。
    3. `demo_time_repaint` 模式下，任何有 `main_draw_list_id` 的通道都被强制标记为脏。
  - **阶段二：拓扑排序**
    1. 遍历所有脏通道，根据父类型决定插入位置：
       - `Window` 或 `Xr`：附加到末尾（顶层通道）。
       - `DrawPass(dep_of_pass_id)`：在 `passes_todo` 中查找父通道位置，将子通道插入父通道之前（确保父先绘制）。
       - `None`：插入到最前面。
    2. 如果依赖的父通道不在列表中，子通道也追加到末尾。
    3. 末尾重置 `demo_time_repaint = false`。
- **设计要点**：结果 `passes_todo` 保证依赖顺序——父通道总在子通道之前。

### `need_redrawing(&self) -> bool`

```rust
pub(crate) fn need_redrawing(&self) -> bool
```

- **实现逻辑**：委托给 `self.new_draw_event.will_redraw()`，检查是否有待处理的绘制事件。
- **用途**：事件循环决策——如果返回 `true`，则不等待新事件直接进入绘制帧。

---

## 二、网络运行时事件分发

### `dispatch_network_runtime_events(&mut self)`

```rust
pub(crate) fn dispatch_network_runtime_events(&mut self)
```

- **职责**：从网络通道接收异步消息并分发为内部事件。
- **实现逻辑**：
  1. 循环调用 `self.net.try_recv()` 接收所有待处理的网络响应（非阻塞）。
  2. 对每个响应，首先检查是否是 Studio 协议消息（通过 `consume_studio_socket_response` 解析），如果是则通过 `dispatch_studio_msg` 直接分发。
  3. 如果是 WebSocket 事件（`WsOpened`、`WsMessage`、`WsClosed`、`WsError`），调用 `handle_script_web_socket_event` 将消息路由到脚本层的 WebSocket 处理器。
  4. HTTP 相关响应（`HttpResponse`、`HttpStreamChunk` 等）暂不处理，仅收集到 `responses` 列表。
  5. 收集完成后，如果有非 Studio 消息，调用 `handle_script_network_events` 将原始网络响应传递给脚本层，再触发 `Event::NetworkResponses` 事件让 Rust 层处理器可以响应。
- **设计要点**：Studio 协议消息直接从响应流中剥离并优先分发，不进入脚本层事件队列，降低了远程调试工具的延迟。

---

## 三、Studio 远程控制协议响应

### `take_studio_screenshot_request_ids(&mut self, kind_id: u32) -> Vec<u64>`

```rust
pub(crate) fn take_studio_screenshot_request_ids(&mut self, kind_id: u32) -> Vec<u64>
```

- **实现逻辑**：
  1. 使用 `retain` 方法遍历 `self.screenshot_requests` 列表。
  2. 对每个请求，如果 `kind_id` 匹配，将 `request_id` 存入返回向量并从列表中移除。
  3. 不匹配的请求保留在原列表中。
  4. 返回所有匹配的 request_id。
- **用途**：在截屏准备就绪后，取出特定窗口类型的所有待处理请求。

### `send_studio_screenshot_response(request_ids, width, height, png)`

```rust
pub(crate) fn send_studio_screenshot_response(request_ids: Vec<u64>, width: u32, height: u32, png: Vec<u8>)
```

- **实现逻辑**：
  1. 如果 `request_ids` 为空，直接返回。
  2. 构造 `AppToStudio::Screenshot(ScreenshotResponse { ... })` 消息。
  3. 通过 `Cx::send_studio_message` 发送给 Studio 前端。
- **用途**：截屏编码完成后，将 PNG 数据回传给 Studio。

### `queue_studio_run_view_frame_request(&mut self, request)`

```rust
pub(crate) fn queue_studio_run_view_frame_request(&mut self, request: RunViewFrameRequest)
```

- **实现逻辑**：
  1. 保留已有的所有请求中 `window_id` 与新请求不重复的项（移除旧请求）。
  2. 将新请求加入列表尾部。
  3. 调用 `self.redraw_all()` 触发重绘，以便尽快生成帧数据。
- **用途**：Studio 请求远程应用的帧数据时，替换并更新请求。

### `take_studio_run_view_frame_request(&mut self, window_id: usize) -> Option<RunViewFrameRequest>`

```rust
pub(crate) fn take_studio_run_view_frame_request(&mut self, window_id: usize) -> Option<RunViewFrameRequest>
```

- **实现逻辑**：
  1. 如果当前有正在进行的异步编码（`run_view_frame_encode_in_flight`），返回 `None` 表示忙。
  2. 使用 `rposition` 从后向前搜索与 `window_id` 匹配的请求（取最新的）。
  3. 使用 `swap_remove` 移除并返回该请求。
  4. 未找到返回 `None`。
- **设计要点**：从后向前搜索优先使用最新请求，跳过旧的过期请求。

### `encode_studio_run_view_frame_async(&mut self, request, width, height, rgba)`

```rust
pub(crate) fn encode_studio_run_view_frame_async(&mut self, request, width, height, rgba)
```

- **实现逻辑**：
  1. 如果 `run_view_frame_encode_in_flight` 为 `true`，直接返回（避免同时进行多个编码）。
  2. 设置 `run_view_frame_encode_in_flight = true` 加锁。
  3. 通过 `self.run_view_frame_results.sender()` 获取回调 sender。
  4. 调用 `self.spawn_thread` 在后台线程中执行：
     a. 调用 `prepare_studio_run_view_rgba` 检查/缩放 RGBA 数据。
     b. 调用 `encode_rgba_as_png` 编码为 PNG。
     c. 构造 `RunViewFrameData` 包含窗口 ID、帧 ID 和 PNG 数据。
     d. 通过 sender 将结果发送回主线程。
  5. 主线程在后续的 `flush_studio_run_view_frame_results` 中消费结果。
- **设计要点**：PNG 编码在后台线程执行，不阻塞主线程的事件循环和绘制。

### `prepare_studio_run_view_rgba(request, width, height, rgba) -> Result<(u32, u32, Vec<u8>), String>`

- **实现逻辑**：
  1. 计算期望的缓冲区大小 `width * height * 4`，与实际大小比较。不一致返回错误。
  2. 从请求中取出目标宽高（最小为 1）。
  3. 如果原始尺寸与目标尺寸一致，直接使用原始缓冲区。
  4. 否则执行最近邻缩放：对目标中的每个像素，通过 `src_x = x * width / target_width` 和 `src_y = y * height / target_height` 计算对应源像素位置，逐像素复制 RGBA 值。
  5. 返回缩放宽、高和数据。
- **设计要点**：最近邻插值（非双线性）以保证性能——RunView 帧通常不需要高质量缩放。

### `flush_studio_run_view_frame_results(&mut self)`

```rust
pub(crate) fn flush_studio_run_view_frame_results(&mut self)
```

- **实现逻辑**：
  1. 循环非阻塞接收 `self.run_view_frame_results` 通道中的编码结果。
  2. 接收到结果后，将 `run_view_frame_encode_in_flight` 设回 `false` 释放锁。
  3. 成功结果通过 `Cx::send_studio_message` 发送为 `AppToStudio::RunViewFrame`。
  4. 失败结果记录错误日志。
  5. 通道为空时退出循环。
- **用途**：在主线程的事件循环中调用，消费后台线程的编码结果。

---

## 四、Widget 树查询与快照

### `send_studio_widget_tree_dump_response(&mut self, request_id: u64)`

- **实现逻辑**：
  1. 将 `request_id` 推入 `self.widget_tree_dump_requests` 队列。
  2. 如果当前在绘制事件中，延迟到绘制结束后处理。
  3. 否则立即调用 `try_send_studio_widget_tree_dump_responses`。

### `send_studio_widget_snapshot_response(&mut self, request_id: u64)`

- 逻辑同上，使用 `widget_snapshot_requests` 队列和 `try_send_studio_widget_snapshot_responses`。

### `send_studio_widget_query_response(&self, request_id, query)`

- **实现逻辑**：
  1. 调用 `self.widget_query_callback` 闭包执行查询。
  2. 将结果封装为 `AppToStudio::WidgetQuery(WidgetQueryResponse { ... })` 发送。
  3. 如果未设置回调，返回空 rects 列表。

### `widget_tree_dump_ready(dump: &str) -> bool`

```rust
fn widget_tree_dump_ready(dump: &str) -> bool
```

- **实现逻辑**：
  1. 逐行解析 dump 文本。
  2. 跳过以 `W` 开头的行（元数据行）。
  3. 对每行取最后两个 token 分别作为高和宽，如果都能解析为非负整数且宽或高 > 0，返回 `true`。
  4. 所有行检查完未找到可见 widget 则返回 `false`。
- **用途**：判断 widget 树是否已完成首次布局和绘制，避免返回空或残缺的 dump。

### `widget_snapshot_ready(widgets: &[WidgetSnapshot]) -> bool`

- **实现逻辑**：检查是否有任何 widget 满足 `visible && width > 0 && height > 0`。
- **用途**：判断快照是否就绪。

### `try_send_studio_widget_tree_dump_responses(&mut self)`

- **实现逻辑**：
  1. 检查是否有待发送的 dump 请求，没有则返回。
  2. 如果当前在绘制事件中（`self.in_draw_event`），延迟发送。
  3. 调用 `self.widget_tree_dump_callback` 生成 dump。
  4. 使用 `widget_tree_dump_ready` 检查 dump 是否有效。
  5. 有效则取出所有待处理 request_id，逐个发送 `AppToStudio::WidgetTreeDump`。

### `try_send_studio_widget_snapshot_responses(&mut self)`

- 逻辑同上，使用 `widget_snapshot_callback` 生成快照数据。

---

## 五、`dispatch_studio_msg` — Studio 消息主分发器

```rust
pub fn dispatch_studio_msg(
    &mut self,
    msg: StudioToApp,
    window_id: WindowId,
    pos: DVec2,
) -> bool
```

这是 `cx_shared.rs` 中最核心的方法，处理来自 Studio 远程控制的全部消息。返回 `true` 表示收到 `Kill` 命令，调用者应关闭应用。

### 鼠标事件处理

- **`MouseDown`**：
  1. 从消息中取出绝对坐标，减去 `pos` 偏移量得到相对于窗口的坐标。
  2. 将 `button_raw_bits` 转换为 `MouseButton`。
  3. 调用 `self.fingers.process_tap_count` 跟踪连击次数。
  4. 调用 `self.fingers.mouse_down` 更新手指状态。
  5. 触发 `Event::MouseDown` 事件。
- **`MouseMove`**：
  1. 构造 `MouseMoveEvent`。
  2. 触发事件。
  3. 调用 `fingers.cycle_hover_area` 更新悬停区域。
  4. 调用 `fingers.switch_captures` 切换手指捕获状态。
- **`MouseUp`**：
  1. 构造 `MouseUpEvent` 并触发。
  2. 调用 `fingers.mouse_up` 释放手指。
  3. 更新悬停区域。
  4. 发送键盘焦点矩形响应（因为点击可能改变了焦点）。
- **`Scroll`**：构造 `ScrollEvent` 并触发，包含滚动偏移量、是否来自鼠标等信息。

### 键盘事件处理

- **`KeyDown`**：先调用 `self.keyboard.process_key_down(e)` 更新键盘状态，再触发 `Event::KeyDown`。
- **`KeyUp`**：先调用 `self.keyboard.process_key_up(e)`，再触发 `Event::KeyUp`。

### 文本事件处理

- **`TextInput`**：直接触发 `Event::TextInput`，包含输入的文字内容。
- **`TextCopy`**：
  1. 创建 `Rc<RefCell<Option<String>>>` 响应通道。
  2. 触发 `Event::TextCopy`，widget 树处理后将剪贴板内容写入响应通道。
  3. 如果响应通道中有内容，发送 `AppToStudio::SetClipboard` 回 Studio。
- **`TextCut`**：逻辑同上，区别在于触发 `Event::TextCut`。

### Studio 工具请求

- **`Screenshot`**：将请求加入 `screenshot_requests` 列表并触发重绘。
- **`RunViewFrameRequest`**：调用 `queue_studio_run_view_frame_request`。
- **`WidgetTreeDump`**：调用 `send_studio_widget_tree_dump_response`。
- **`WidgetQuery`**：调用 `send_studio_widget_query_response`。
- **`WidgetSnapshot`**：调用 `send_studio_widget_snapshot_response`。

### 生命周期与自定义

- **`Kill`**：触发 `Event::Shutdown` 并返回 `true`。
- **`Custom(data)`**：触发 `Event::Custom(data)` 供应用层处理自定义消息。

### 热重载与无操作

- **`LiveChange { file_name, content }`**：将文件变更加入 `script_data.live_reload.queue_file_change`，等待热重载系统处理。
- **`KeepAlive` / `None`**：无操作。
- **`Tick` / `Swapchain` / `WindowGeomChange` / `TweakRay`**：无操作（这些由 stdin 通道的调用者处理）。

---

## 六、`poll_control_channel` — 控制通道轮询

```rust
pub fn poll_control_channel(&mut self)
```

- **实现逻辑**：
  1. 锁定 `CONTROL_CHANNEL` 全局变量。
  2. 如果有 receiver，使用 `try_iter` 非阻塞收集所有待处理消息。
  3. 为每条消息调用 `dispatch_studio_msg`，使用窗口 ID 0（虚拟窗口）和原点坐标。
- **用途**：从全局控制通道接收外部注入的 Studio 消息（用于无 stdin 连接的场景）。

---

## 七、`run_live_edit_if_needed` — 热重载分发

```rust
pub(crate) fn run_live_edit_if_needed(&mut self, _backend: &str)
```

- **职责**：检查并执行热重载、脚本重应用操作。
- **实现逻辑**（三步分支）：

  1. **`LiveEditTrigger::FileChange`**（DSL 变更）：
     - 调用 `self.draw_shaders.reset_for_live_reload()` 重置着色器缓存。
     - 清除 `pending_script_reapply`。
     - 触发 `Event::LiveEdit`（重新加载 script_mod! 块）。
     - 如果触发了 `pending_script_reapply`（例如偏好重广播），立即在当前 tick 执行 `Event::ScriptReapply`。

  2. **`LiveEditTrigger::Manual`**（手动触发，如安全区域变化）：
     - 清除 `pending_script_reapply`（script_mod 重跑会覆盖堆覆盖）。
     - 触发 `Event::LiveEdit`。
     - 不执行 `ScriptReapply`——延迟到下一 tick，避免旋转动画中反复 Apply 导致的卡顿。

  3. **`LiveEditTrigger::None`** + `pending_script_reapply`：
     - 触发 `Event::ScriptReapply`，保持堆覆盖不变。
     - 不清空 script_mod 块。

---

## 八、事件分发基础设施

### `inner_call_event_handler(&mut self, event: &Event)`

```rust
pub(crate) fn inner_call_event_handler(&mut self, event: &Event)
```

- **职责**：调用实际的用户事件处理器。
- **实现逻辑**：
  1. 自增 `self.event_id`。
  2. 如果与 Studio 有 WebSocket 连接且未处于 stdout 模式，或者启用了本地性能分析：
     a. 记录开始时间戳。
     b. 取出 `self.event_handler`（可选闭包），调用它处理事件。
     c. 放回 event_handler。
     d. 记录结束时间戳。
     e. 发送 `AppToStudio::EventSample`，包含事件类型、开始时间、结束时间。如果是 Timer 事件，还包含 timer_id。
  3. 否则直接调用 event_handler（无性能采样）。
  4. 事后处理 Studio 协议相关：
     a. 调用 `try_send_studio_widget_tree_dump_responses` 发送待处理的 dump 响应。
     b. 调用 `try_send_studio_widget_snapshot_responses` 发送待处理的快照响应。
  5. 清理 widget 查询失效事件：如果 `self.event_id > widget_query_invalidation_event + 1`，说明失效已传播完整 widget 树，清除失效标记。

### `inner_key_focus_change(&mut self)`

```rust
fn inner_key_focus_change(&mut self)
```

- **实现逻辑**：
  1. 调用 `self.keyboard.cycle_key_focus_changed()` 检查键盘焦点是否变化。
  2. 如果有变化，取出 `(prev, focus)` 两个 Area。
  3. 触发 `Event::KeyFocus(KeyFocusEvent { prev, focus })`。

### `handle_triggers(&mut self)`

```rust
pub fn handle_triggers(&mut self)
```

- **实现逻辑**：
  1. 循环直到 `self.triggers` 列表为空。
  2. 每次循环将 `self.triggers` 与空 HashMap 交换（清空原列表）。
  3. 调用 `inner_call_event_handler` 触发 `Event::Trigger`。
  4. 调用 `inner_key_focus_change` 处理焦点变化。
  5. 超过 100 次迭代则报"触发器反馈循环"错误并终止。
- **用途**：处理 widget 树中产生的触发器事件，触发器处理器可能产生新的触发器，因此需要循环处理。

### `handle_actions(&mut self)`

```rust
pub fn handle_actions(&mut self)
```

- 逻辑与 `handle_triggers` 相同，但处理 `self.new_actions` 列表和 `Event::Actions`。
- 超过 100 次迭代时额外打印 `self.new_actions` 内容以辅助调试。

### `handle_pending_window_geom_changes(&mut self)`

```rust
pub fn handle_pending_window_geom_changes(&mut self)
```

- **实现逻辑**：
  1. 循环直到 `self.pending_window_geom_changes` 为空。
  2. 每次交换取走所有待处理事件。
  3. 逐个触发 `Event::WindowGeomChange`，并在每个事件后处理焦点变化。
  4. 超过 100 次迭代报"WindowGeomChange feedback loop"错误。
- **用途**：处理在事件处理期间产生的窗口几何变化事件（例如 `set_window_dpi_override`）。

### `call_event_handler(&mut self, event: &Event)`

```rust
pub(crate) fn call_event_handler(&mut self, event: &Event)
```

- **职责**：完整的事件处理流水线入口。
- **实现逻辑**（严格顺序）：
  1. 如果事件是 `PermissionResult`，先处理相机权限。
  2. 调用 `inner_call_event_handler` 执行主事件处理。
  3. 处理事件处理期间产生的窗口几何变化（`handle_pending_window_geom_changes`）。
  4. 处理键盘焦点变化（`inner_key_focus_change`）。
  5. 处理触发器（`handle_triggers`）。
  6. 处理动作（`handle_actions`）。
  7. 处理脚本任务队列（`handle_script_tasks`）。
  8. 再次处理窗口几何变化、焦点变化、触发器、动作（因为脚本回调可能产生新的这些事件）。
  9. **设计要点**：动作/触发器/脚本任务的分发形成一个稳定扩展的流水线而非递归，可以有效地检测和打破反馈循环。

### `call_draw_event(&mut self, time: f64)`

```rust
pub(crate) fn call_draw_event(&mut self, time: f64)
```

- **实现逻辑**：
  1. 交换 `self.new_draw_event` 取出待处理绘制事件。
  2. 设置 `self.in_draw_event = true`。
  3. 调用 `call_event_handler` 触发 `Event::Draw`。
  4. 设置 `self.in_draw_event = false`。
  5. 如果连接了 Studio WebSocket，发送待处理的 widget tree dump 和 snapshot。

### `call_next_frame_event(&mut self, time: f64)`

```rust
pub(crate) fn call_next_frame_event(&mut self, time: f64)
```

- **实现逻辑**：
  1. 交换 `self.new_next_frames` 取出待处理的下一帧请求集合。
  2. 调用 `self.performance_stats.process_frame_data` 记录帧性能数据。
  3. 调用 `call_event_handler` 触发 `Event::NextFrame`，包含请求集合、时间和重绘序号。
- **用途**：`request_next_frame` 注册的回调在下一帧绘制前执行，用于持续动画或后台更新。

---

## 九、PNG 编码

### `encode_rgba_as_png(width: u32, height: u32, rgba: &[u8]) -> Result<Vec<u8>, String>`

```rust
pub fn encode_rgba_as_png(width: u32, height: u32, rgba: &[u8]) -> Result<Vec<u8>, String>
```

- **实现逻辑**：
  1. 使用 `makepad_zune_png::PngEncoder` 库。
  2. 配置编码选项：宽度、高度、8位深度、RGBA 色彩空间。
  3. 创建编码器实例，传入 RGBA 数据和选项。
  4. 调用 `encode` 将 PNG 数据写入输出缓冲区。
  5. 使用 `map_err` 将编码器错误转为 `String`。
- **与 headless 的 `encode_png_rgba` 共享相同逻辑但独立存在**，因为 headless 版本被 `cfg(headless)` 隔离。

---

## 十、总结

`cx_shared.rs` 是 Makepad 平台层中体积最大、功能最集中的文件。它将以下核心能力统一到 `Cx` 上：

| 功能域 | 方法 | 说明 |
|--------|------|------|
| 绘制管理 | `repaint_windows`, `any_passes_dirty`, `compute_pass_repaint_order` | 脏标记传播与拓扑排序 |
| Studio 远程 | `dispatch_studio_msg` 及辅助方法 | 输入事件转发、截屏、widget 查询 |
| 帧编码 | `encode_studio_run_view_frame_async`, `encode_rgba_as_png` | 异步 PNG 编码 |
| 热重载 | `run_live_edit_if_needed` | 三种 LiveEdit 触发模式 |
| 事件流水线 | `call_event_handler`, `handle_triggers`, `handle_actions` | 带反馈检测的分层事件处理 |

通过将所有跨平台共享逻辑集中在此文件，各平台后端只需关注 OS 特定的事件循环和窗口管理，事件处理、远程控制、热重载等复杂度被完全隔离。
