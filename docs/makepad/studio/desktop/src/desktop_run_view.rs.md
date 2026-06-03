# `desktop_run_view.rs` — Studio 远程运行视图

## 文件作用

实现应用的远程运行视图，支持两种运行模式：
1. **本地 Swapchain**（进程内 gateway）：通过共享内存交换链传输帧数据，用于桌面端同进程运行
2. **远程帧**（WebSocket）：通过 PNG/JPEG 编码传输帧数据，用于远程/移动端运行

同时实现输入可视化（点击/键盘焦点动画）和光标同步。

## script_mod 定义

注册 `DesktopRunView` widget，包含三个绘制层：
- `draw_bg`：背景（`theme.color_bg_container`）
- `draw_app`：应用帧纹理（支持 packed header 和 y-flip 两种格式）
- `draw_ai_viz`：输入可视化动画（圆点 ripple 和圆角矩形 focus 框）
- `no_fb_view`：无帧缓冲时的占位视图

### Shader 实现

**应用帧 shader** (`draw_app`):
- `tex: texture_2d(float)` — 帧纹理
- `tex_scale` / `tex_size` / `host_dpi_factor` — 纹理缩放参数
- `y_flip` — Linux 下需要垂直翻转
- `packed_header` — 区分直接纹理和 packed header 格式
- packed header 格式：从纹理前两个像素解码帧尺寸，用于动态缩放

**输入可视化 shader** (`draw_ai_viz`):
- `dot_radius` / `dot_alpha` — 圆点不透明度动画
- `ripple_radius` / `ripple_alpha` — 水波扩散动画
- `shape_kind` — 0 = 圆形（点击），1 = 圆角矩形（文本输入/回车）
- `corner_radius` / `stroke_width` — 矩形参数

## 关键数据结构

### `RunTarget`
```rust
struct RunTarget {
    build_id: QueryId,
    window_id: usize,  // 窗口 ID（默认 0 用于 stdin-loop app）
}
```

### `InputVizEvent`
```rust
struct InputVizEvent {
    kind: RunViewInputVizKind,  // 点击/文本/回车
    pos: Vec2d,
    size: Option<Vec2d>,        // 可选焦点矩形尺寸
}
```

### `PendingRemoteDecode`
异步图像解码等待状态。

## `DesktopRunView` 结构体

主要字段：
- `draw_bg` / `draw_app` / `draw_ai_viz` / `no_fb_view` — 绘制层
- `current_target: Option<RunTarget>` — 当前运行目标
- `swapchain` / `last_swapchain_with_completed_draws` — 交换链管理
- `pending_draw` — 待应用的帧绘制
- `remote_*` 系列字段 — 远程帧状态（frame_id, path, decode pending 等）
- `ai_viz_*` 系列字段 — 输入可视化状态
- `aux_chan_host_endpoint` — Linux aux channel 端点（仅 Linux）

## ScriptHook

`on_after_new`：初始化纹理为 null texture，启动 8ms 间隔的 tick 定时器，设置 `packed_header = 1.0`。

## 核心方法

### 运行目标管理

**`set_target`**: 切换/清除运行目标时重置所有状态（cursor, swapchain, ai_viz, remote_mode 等）。如果设置了新目标，保持 240 帧的重绘计数以完成 bootstrap。

**`set_run_target`**: 调用 `set_target` 后，在 Linux 上启动 aux channel 监听。

**`clear_run_target`**: 清除运行目标。

**`rebootstrap_after_app_ready`**: 在 app 发送 `RunViewCreated` 后重新发送 bootstrap 消息。设置 `app_ready_for_swapchain = true` 并重置 bootstrap 计数。

### 帧路由

**`set_presentable_draw`**: 尝试立即呈现本地 swapchain 帧；如果 swapchain 不存在则保存为 pending。

**`set_remote_frame`**: 接收远程帧数据（PNG/JPEG），存入异步图像缓存后设置纹理。

**`apply_remote_texture`**: 将解码后的纹理绑定到 `draw_app` 并设置纹理参数。

**`request_remote_frame_if_needed`**: 每帧检查是否需要请求新的远程帧。

### Swapchain 管理

**`apply_presentable_draw_to_quad`**: 将 swapchain 图像绑定到 `draw_app` 纹理。

**`try_present_draw`**: 尝试在当前或备用的 swapchain 上呈现帧。

**`ensure_swapchain_for_rect`**: 根据窗口尺寸和 DPI 创建或重建交换链。Linux 上使用确切尺寸，其他平台使用 `next_power_of_two` 对齐。

### Bootstrap 消息

**`build_bootstrap_msgs`**: 构建发给子 app 的启动消息序列：
1. `WindowGeomChange` — 窗口几何信息
2. `Swapchain` — 共享交换链描述符（app_ready 后发送）

### 输入可视化

**`show_input_viz`**: 从 hub 接收输入可视化事件。区分点击（用坐标）和文本输入/回车（用焦点矩形）。支持事件排队。

**`start_input_viz` / `enqueue_or_start_input_viz`**: 开始新的可视化动画，或在已有动画时排队。

**`set_input_focus_rect`**: 设置输入焦点矩形位置，释放积压的焦点可视化事件。

### 事件处理（Widget trait）

**处理的事件**：
- `Timer`（tick）：发送 bootstrap 消息和帧请求
- `FingerDown` → `MouseDown`（带 button, position, time）
- `FingerMove` → `MouseMove`
- `FingerHoverIn/Over` → 设置远程光标 + `MouseMove`
- `FingerHoverOut` → 重置光标
- `FingerUp` → `MouseUp`
- `FingerScroll` → `Scroll`
- `TextInput` → `TextInput`
- `KeyDown` / `KeyUp` → `KeyDown` / `KeyUp`
- `TextCopy` / `TextCut` → 转发

所有输入事件通过 `StudioToApp` 消息转发到目标 app。

### `draw_walk`
1. 设置 DPI 感知的 turtle
2. 绘制背景
3. 重新设置 target（确保状态同步）
4. 更新 swapchain
5. 尝试呈现 pending draw
6. 绘制应用帧纹理
7. 绘制输入可视化动画（每帧递减计数器，显示 ripple 动画）
8. 如果无帧缓冲，显示 "no framebuffer" 占位
9. 更新面积为 `draw_app` 的 area
10. 如果有焦点，设置 IME 位置

## DesktopRunViewRef 方法

通过 `borrow_mut` 访问内部 `DesktopRunView` 的公开接口：
- `set_run_target` / `set_presentable_draw` / `set_remote_frame`
- `clear_run_target` / `set_remote_cursor` / `show_input_viz`
- `set_input_focus_rect` / `rebootstrap_after_app_ready`
