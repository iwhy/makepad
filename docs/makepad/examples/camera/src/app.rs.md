# `examples/camera/src/app.rs`

Makepad Camera 示例的核心文件。展示了如何在 Makepad 中集成摄像头预览，支持三种模式：无摄像头、Texture 模式和 NativePreview 模式。通过 `script_mod!` DSL 定义 UI，通过 Rust 的 `MatchEvent` 和 `AppMain` trait 处理摄像头生命周期和用户交互。

## 核心概念

- **三种摄像头模式**: `NoCamera`（关闭）、`Texture`（软件纹理模式）、`NativePreview`（原生预览模式）
- **摄像头选择策略**: 优先级 NV12 > YUY2 > YUV420，且优先选择 ≤1080p 的 NV12
- **权限管理**: 通过 Makepad 的权限系统请求摄像头权限
- **状态驱动**: 使用 `pending_mode_switch` 标志和 `drive_mode` 状态机管理模式切换

## `script_mod!` UI 定义（行 6-92）

```rust
script_mod! {
    use mod.prelude.widgets.*
    use mod.widgets.*

    startup() do #(App::script_component(vm)){
        ui: Root{
            main_window := Window{
```

UI 结构为：一个 `Window` 包含标题 "Camera Home"、三个模式选择按钮（no-camera/texture/nativepreview）、状态信息标签和三个摄像头显示区域（占位符、原生预览、纹理预览），通过 `visible` 属性控制显示。

### 关键 UI 组件

- **mode_row**: 三个模式按钮，`width: Fill` 均匀分布
- **mode_label / rotation_label / status_label**: 显示当前模式、YUV 旋转角度和摄像头状态
- **camera_placeholder**: 无摄像头时的默认占位视图
- **camera_native_host / camera_texture_host**: 分别包含 `Video` 组件，可见性互斥
- **Video**: 使用 Makepad 的 `Video` 组件进行摄像头画面展示，设置 `autoplay: false` 和 `show_controls: false`

## App 结构体（行 316-330）

```rust
#[derive(Script, ScriptHook)]
pub struct App {
    #[live] ui: WidgetRef,
    #[rust] desired_mode: CameraHomeMode,
    #[rust] pending_mode_switch: bool,
    #[rust] camera_permission: Option<PermissionStatus>,
    #[rust] camera_choice: Option<CameraChoice>,
    #[rust] last_yuv_rotation_steps: f32,
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `ui` | `WidgetRef` | 根 UI 组件引用，`#[live]` 标记使其可从脚本访问 |
| `desired_mode` | `CameraHomeMode` | 用户选择的模式（默认为 `NoCamera`） |
| `pending_mode_switch` | `bool` | 是否有挂起的模式切换操作 |
| `camera_permission` | `Option<PermissionStatus>` | 摄像头权限状态 |
| `camera_choice` | `Option<CameraChoice>` | 选中的摄像头设备和格式 |
| `last_yuv_rotation_steps` | `f32` | 最后一次 YUV 纹理更新的旋转步数 |

## CameraHomeMode 枚举（行 332-356）

```rust
enum CameraHomeMode {
    NoCamera,       // 关闭摄像头
    Texture,        // 软件纹理预览模式
    NativePreview,  // 原生预览模式
}
```

### 方法

| 方法 | 说明 |
|------|------|
| `label()` | 返回模式名称字符串，用于 UI 显示 |
| `to_preview_mode()` | 转换为 `VideoCameraPreviewMode` 枚举。当 `NoCamera` 时返回 `Texture`（无害默认值） |

## CameraChoice 结构体（行 358-367）

存储选中的摄像头设备信息：

| 字段 | 说明 |
|------|------|
| `input_id` | 摄像头输入设备 ID |
| `format_id` | 选定格式的 ID |
| `name` | 设备名称 |
| `width/height` | 分辨率 |
| `pixel_format` | 像素格式（NV12、YUY2、YUV420 等） |
| `frame_rate` | 帧率（可选） |

## 关键方法

### `set_status`（行 95-97）

更新状态标签文本。通过 `ids!` 宏引用 UI 组件。

### `set_preview_mode_visible`（行 99-109）

根据模式切换三个容器视图的可见性：`None` 显示占位符，`Some(NativePreview)` 显示原生预览，`Some(Texture)` 显示纹理预览。

### `pick_camera_choice`（行 125-212）

摄像头选择算法的核心逻辑——**三级递进选择**：

1. **Pass 1（行 166-176）**: 仅选择 NV12 格式且分辨率 ≤ 1920×1080 的设备。这是 iOS 纹理路径的首选
2. **Pass 2（行 179-188）**: 如果 Pass 1 无结果，选择任意 NV12 格式
3. **Pass 3（行 191-200）**: 如果仍然无结果，选择其他受支持的 YUV 格式（YUY2、YUV420）

**`better` 函数（行 144-161）**: 在两个格式间择优：
- 优先 pixel_rank 更高的（NV12 > YUY2 > YUV420 > 其他）
- 其次分辨率更高的（像素数更多）
- 最后帧率更高的

### `choose_mode`（行 214-221）

用户点击按钮后的入口：设置新模式、标记挂起切换、更新标签、驱动状态机。

### `drive_mode`（行 223-313）

核心**状态机**，分模式处理：

**NoCamera 模式**:
- 如果两个 Video 都已空闲 → 标记切换完成
- 否则依次清理两个 Video 的资源

**Texture 或 NativePreview 模式**:
1. 先将所有预览视图隐藏
2. 检查摄像头权限，未授权则等待
3. 检查摄像头设备信息，未就绪则等待
4. 根据模式选择目标 Video 和另一个 Video
5. 如果另一个 Video 仍活跃，先停止清理它
6. 如果目标 Video 尚未就绪，启动摄像头：`set_camera_preview_mode` → `set_source_camera` → `begin_playback`
7. 更新状态信息（设备名、分辨率、像素格式、帧率）

## 事件处理

### `MatchEvent::handle_actions`（行 369-381）

处理按钮点击事件：

```rust
if self.ui.button(cx, ids!(no_camera_btn)).clicked(actions) {
    self.choose_mode(cx, CameraHomeMode::NoCamera);
}
```

三个模式按钮分别触发对应的 `choose_mode` 调用。

### `AppMain::handle_event`（行 383-477）

处理整个应用事件循环：

| 事件 | 处理 |
|------|------|
| `Event::Startup` | 请求摄像头权限、初始化 Video 输入、更新 UI 标签 |
| `Event::VideoInputs(ev)` | 选择最佳摄像头格式、如果没有可用摄像头则显示提示 |
| `Event::PermissionResult` | 处理权限结果：已授权则驱动模式切换，拒绝则显示提示 |
| `Event::VideoPlaybackPrepared` | 视频就绪后显示原生预览、更新状态 |
| `Event::VideoTextureUpdated` | 更新 YUV 旋转角度显示、显示纹理预览 |
| `Event::VideoPlaybackResourcesReleased` | 资源释放后继续驱动状态机 |
| `Event::VideoDecodingError` | 显示错误信息 |

## 关键模式

### 1. 状态驱动切换

使用 `pending_mode_switch` 标志避免在摄像头状态过渡中重复操作。`drive_mode` 被设计为幂等的——每次事件到达时再次调用，直到所有条件满足。

### 2. 安全清理

切换模式时先清理旧模式的 Video 资源（通过 `stop_and_cleanup_resources`），等待 `VideoPlaybackResourcesReleased` 事件后再启动新模式。

### 3. 循环依赖打破

`drive_mode` 可能通过启动操作触发新的 `VideoPlaybackPrepared`/`VideoTextureUpdated` 事件，这些事件又调用 `drive_mode`，形成自然的推进循环。

## 总结

这个示例完整展示了 Makepad 的摄像头集成流程：
- 使用 `Video` widget 进行摄像头预览
- 权限请求与回调处理
- 摄像头设备枚举与格式选择
- 状态驱动的模式切换管理
- YUV 纹理旋转信息处理
