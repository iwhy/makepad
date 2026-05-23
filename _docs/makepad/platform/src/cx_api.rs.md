# `cx_api.rs` — Cx 公共 API 方法

## 概述

`cx_api.rs` 是 `Cx` 结构的公共 API 方法实现文件（约 1781 行）。定义了 `CxOsApi` trait（操作系统抽象接口）、`CxOsOp` 枚举（平台操作命令）、`CxSystemBrowser`（系统浏览器控制）、`XrFrameCpuBreakdown`（XR 帧性能分解），以及 `Cx` 的大量方法，覆盖窗口管理、纹理上传、着色器编译、事件处理、定时器、权限、文本 IME、剪贴板、拖放、视频播放、音频播放、文件对话框、网络请求、XR 等所有功能。

---

## 类型

### `OpenUrlInPlace`（第 37-40 行）

```rust
pub enum OpenUrlInPlace { Yes, No }
```

### `CxThreadPriority`（第 42-49 行）

线程优先级枚举：`Normal`、`Utility`、`Background`、`Idle`。

### `XrFrameCpuBreakdown`（第 51-85 行）

XR 帧 CPU 时间分解结构体，包含从 wait_frame 到 end_frame 的 23 个细分阶段（单位：ms），以及重绘阶段的纹理上传、几何上传、绘制调用等统计数据。用于 XR 性能分析和优化。

### `SystemBrowserId` / `CxSystemBrowser`（第 87-147 行）

- `SystemBrowserId(LiveId)` — 系统浏览器的标识符
- `CxSystemBrowser<'a>` — 系统浏览器操作句柄，提供 `spawn()`、`update()`、`detach()`、`set_url()`、`history_go()`、`close()` 方法。所有操作都通过向 `cx.platform_ops` 推送 `CxOsOp` 延迟执行

---

## `CxOsApi` trait（第 149-220 行）

操作系统抽象接口，定义了 Cx 需要平台层提供的能力：

| 方法 | 说明 |
|------|------|
| `init_cx_os()` | 初始化 Cx 操作系统层 |
| `spawn_thread(F)` | 创建新线程执行闭包 |
| `start_stdin_service()` | 启动标准输入服务（macOS 使用） |
| `pre_start() -> bool` | 应用启动前的预处理阶段 |
| `open_url(url, in_place)` | 在系统浏览器中打开 URL |
| `browser_update_url(url, replace)` | 浏览器更新 URL |
| `browser_history_go(delta)` | 浏览器前进/后退 |
| `seconds_since_app_start() -> f64` | 获取从应用启动到现在的秒数 |
| `default_window_size() -> Vec2d` | 默认窗口大小（800x600） |
| `max_texture_width() -> usize` | 最大纹理宽度（4096） |
| `in_xr_mode() -> bool` | 是否处于 XR 模式 |
| `micro_zbias_step() -> f32` | Z 偏移步进值 |
| `xr_render_scale()` / `xr_gpu_frame_time_ms()` / ... | XR 性能查询方法 |
| `xr_frame_cpu_breakdown()` | XR 帧 CPU 时间分解 |
| `xr_display_refresh_rate_hz()` / `xr_effective_frame_rate_hz()` | XR 刷新率 |

---

## `CxOsOp`——平台操作命令枚举（第 238-403 行）

`Cx` 不直接调用平台 API，而是将操作封装为 `CxOsOp` 枚举推入 `platform_ops` 向量，由 OS 层在一次事件循环中批量处理。

**窗口操作：**
- `CreateWindow(WindowId)` — 创建窗口
- `CreatePopupWindow { window_id, parent_window_id, position, size, grab_keyboard }` — 创建弹出窗口
- `ResizeWindow(WindowId, Vec2d)` — 调整窗口大小
- `RepositionWindow(WindowId, Vec2d)` — 移动窗口
- `CloseWindow(WindowId)` — 关闭窗口
- `MinimizeWindow` / `Deminiaturize` / `MaximizeWindow` / `FullscreenWindow` / `NormalizeWindow` / `RestoreWindow` / `HideWindow` — 窗口状态操作
- `HideWindowButtons` / `ShowWindowButtons` — 窗口按钮显示控制
- `SetTopmost(WindowId, bool)` — 置顶/取消置顶
- `SetWindowVisuals(WindowId, WindowVisuals)` — 窗口视觉效果
- `ShowInDock(bool)` — 是否在 dock 中显示
- `SetSystemBarDarkIcons(bool)` — 系统栏图标深色模式（Android）

**输入法操作：**
- `ShowTextIME(Area, Vec2d, TextInputConfig)` — 显示输入法
- `HideTextIME` — 隐藏输入法
- `SyncImeState { text, selection, composition }` — 同步 IME 状态

**光标/定时器/退出：**
- `SetCursor(MouseCursor)` — 设置鼠标光标样式
- `StartTimer { timer_id, interval, repeats }` — 启动定时器
- `StopTimer(u64)` — 停止定时器
- `Quit` — 退出应用

**剪贴板/选择/无障碍：**
- `StartDragging(Vec<DragItem>)` — 开始拖拽
- `UpdateMacosMenu(MacosMenu)` — 更新 macOS 菜单栏
- `ShowClipboardActions { has_selection, rect, keyboard_shift }` — 显示剪贴板操作菜单
- `HideClipboardActions` / `CopyToClipboard(String)` / `SetPrimarySelection(String)`
- `ShowSelectionHandles` / `UpdateSelectionHandles` / `HideSelectionHandles` — 移动端选择手柄
- `AccessibilityUpdate(AccessibilityUpdatePayload)` — 无障碍树更新

**权限：**
- `CheckPermission { permission, request_id }` — 检查权限状态
- `RequestPermission { permission, request_id }` — 请求权限

**网络/HTTP：**
- `HttpRequest { request_id, request }` — 发起 HTTP 请求
- `CancelHttpRequest { request_id }` — 取消 HTTP 请求

**视频播放：**
- `PrepareVideoPlayback(...)` — 准备视频播放
- `AttachCameraNativePreview` / `UpdateCameraNativePreview` / `DetachCameraNativePreview` — 原生相机预览
- `BeginVideoPlayback` / `PauseVideoPlayback` / `ResumeVideoPlayback` / ... — 播放控制
- `SeekVideoPlayback` / `SetVideoVolume` / `SetVideoPlaybackRate` — 播放参数
- `UpdateVideoSurfaceTexture` — 更新视频表面纹理

**音频播放：**
- `PrepareAudioPlayback(...)` — 准备音频播放

**系统浏览器：**
- `SpawnSystemBrowser` / `UpdateSystemBrowser` / `DetachSystemBrowser`
- `SetSystemBrowserUrl` / `SystemBrowserHistoryGo` / `CloseSystemBrowser`

**WebView：**
- `CreateWebView { id, area, texture, url }` / `UpdateWebView` / `CloseWebView`

**文件对话框：**
- `SaveFileDialog(FileDialog)` / `SelectFileDialog(FileDialog)`
- `SaveFolderDialog(FileDialog)` / `SelectFolderDialog(FileDialog)`

**XR：**
- `XrStartPresenting` / `XrStopPresenting` — 开始/停止 XR 展示
- `XrSetRenderScale(f32)` — 设置渲染缩放
- `XrSetLocalAnchor(XrAnchor)` / `XrSetLocalFloor(f32)` — 空间锚点
- `XrAdvertiseAnchor(XrAnchor)` / `XrDiscoverAnchor(u8)` — 锚点分享/发现

---

## `Cx` 方法详解

### 脚本/热重载控制

| 方法 | 说明 |
|------|------|
| `request_script_reapply()` | 请求 `Event::ScriptReapply`，保留运行时 `script_eval!` 覆盖。设置 `pending_script_reapply = true` |
| `request_live_edit()` | 请求 `Event::LiveEdit`，重新运行 `script_mod` + `Apply::Reload`。会清空运行时 widget 状态。只在基本堆值（如 `SAFE_INSET_PAD_TOP`）发生变化且被 `script_mod!` 表达式引用时使用 |

### 安全区域

| 方法 | 说明 |
|------|------|
| `update_safe_inset_script_values(insets)` | 将安全区域值写入脚本堆中的 `SAFE_INSET_PAD_TOP/BOTTOM/LEFT/RIGHT` 变量。获取 `script_vm`，通过 vm.heap.set_value 将 insets.top/bottom/left/right 分别设置到 mod.widgets 命名空间下 |

### XR 查询

| 方法 | 说明 |
|------|------|
| `xr_capabilities()` | 返回 XR 能力引用 |
| `xr_tsdf()` | 返回 XR TSDF 存储 |
| `xr_render_scale()` / `xr_gpu_frame_time_ms()` / `xr_frame_cpu_time_ms()` / ... | 委托到 `CxOsApi` trait 的对应方法 |

### 资源管理

| 方法 | 说明 |
|------|------|
| `geometry_pool_slot_count()` / `geometry_pool_live_count()` | 几何池槽位/活跃数量 |
| `draw_list_pool_slot_count()` / `draw_list_pool_live_count()` | 绘制列表池统计 |
| `texture_pool_slot_count()` / `texture_pool_live_count()` | 纹理池统计 |
| `take_dependency(path)` | 从依赖表中取出数据（消费式，只取一次）。失败时尝试 Android asset 加载 |
| `get_dependency(path)` | 从依赖表中获取数据（克隆式）。失败时尝试 Android asset 加载 |
| `get_resource(handle)` | 通过 ScriptHandle 获取已加载资源数据。Web 平台额外尝试从依赖表回退查找 |
| `get_resource_abs_path(handle)` | 获取资源文件的绝对路径 |
| `get_resource_font_bytes(handle)` | 获取字体资源数据。优先通过 mmap 从本地文件读取，回退到已加载的内存数据（wasm/网络资源使用） |

### 平台信息

| 方法 | 说明 |
|------|------|
| `in_draw_event()` | 是否在绘制事件中 |
| `null_texture()` / `null_cube_texture()` | 返回空纹理/空立方体纹理的克隆 |
| `redraw_id()` | 返回当前重绘 ID |
| `os_type()` | 返回操作系统类型引用 |
| `get_data_dir()` | 获取可写数据目录（Android: filesDir, iOS: Application Support）。平台上无此概念返回 None |
| `in_makepad_studio()` | 是否运行在 Studio 中 |
| `cpu_cores()` | CPU 核心数 |
| `gpu_info()` | GPU 信息引用 |

### 窗口/渲染管理

| 方法 | 说明 |
|------|------|
| `set_window_dpi_override(window_id, dpi_override)` | 运行时设置窗口的 DPI 覆盖。重新计算所有几何度量（inner_size、outer_size、safe_area_insets、chrome buttons），生成 `WindowGeomChangeEvent` 并触发全局重绘 |
| `get_dpi_factor_of(area)` | 获取指定 Area 所在窗口的 DPI 因子。通过 draw_list_id → draw_pass_id → 向上遍历 pass 层级树找到 Window，返回其 dpi_factor |
| `get_window_id_of(area)` | 获取 Area 所属的 WindowId（遍历 pass 层级树） |
| `get_pass_window_id(draw_pass_id)` | 从 draw_pass 向上遍历父链（最多 25 层），找到关联的 WindowId |
| `get_delegated_dpi_factor(draw_pass_id)` | 遍历 pass 层级树，获取委托的 DPI 因子 |
| `redraw_pass_and_parent_passes(draw_pass_id)` | 重绘指定 pass 及其所有父 pass |
| `get_pass_rect(draw_pass_id, dpi)` | 获取 pass 的矩形区域（支持 Area、AreaOrigin、Size 三种 rect 类型） |
| `get_pass_name(draw_pass_id)` | 获取 pass 的调试名称 |
| `repaint_pass(draw_pass_id)` | 标记 pass 的 paint_dirty = true |
| `repaint_pass_and_child_passes(draw_pass_id)` | 递归标记 pass 及所有子 pass 为 paint_dirty |
| `redraw_pass_and_child_passes(draw_pass_id)` | 递归重绘 pass 及所有子 pass 的 draw list |
| `redraw_all()` | 设置全局重绘标志 |
| `redraw_area(area)` | 标记 area 所在 draw list 需要重绘 |
| `redraw_area_in_draw(area)` | 在绘制事件中标记 area 重绘（不限 in_draw_event 标志） |
| `redraw_area_and_children(area)` | 标记 area 及所有子 draw list 重绘 |
| `redraw_list(draw_list_id)` | 标记 draw list 需要重绘（在绘制事件中忽略） |
| `redraw_list_in_draw(draw_list_id)` | 标记 draw list 重绘（内部：不重复添加已在队列中的 list） |
| `redraw_list_and_children(draw_list_id)` | 递归标记 draw list 及其全部子 draw list 重绘 |

### 输入法（IME）

| 方法 | 说明 |
|------|------|
| `show_text_ime(area, pos)` | 在指定位置显示文本输入法（默认配置） |
| `show_text_ime_with_config(area, pos, config)` | 显示文本输入法（自定义配置）。检查 `keyboard.text_ime_dismissed`，未取消时才推入平台操作 |
| `sync_ime_state(text, selection, composition)` | 同步 IME 的文本、选区、组合状态 |
| `hide_text_ime()` | 隐藏文本输入法，重置 dismissed 标志 |
| `text_ime_was_dismissed()` | IME 被平台取消时的回调 |
| `get_ime_area_rect()` | 获取当前 IME 关联的 Area 矩形 |

### Area 管理

| 方法 | 说明 |
|------|------|
| `update_area_refs(old_area, new_area)` | 更新所有对旧 Area 的引用为新区。检查 IME area、手指状态、拖放状态、键盘状态中的旧 area，替换为新 area。返回 new_area |

### 键盘焦点

| 方法 | 说明 |
|------|------|
| `set_key_focus(focus_area)` | 设置键盘焦点到指定 Area |
| `key_focus()` | 获取当前键盘焦点 Area |
| `revert_key_focus()` | 恢复键盘焦点到前一个焦点 |
| `has_key_focus(focus_area)` | 检查指定 Area 是否有键盘焦点 |

### 定时器/帧/触发器

| 方法 | 说明 |
|------|------|
| `new_next_frame()` | 分配新的 `NextFrame` ID，将其加入 `new_next_frames` 集合 |
| `send_trigger(area, trigger)` | 向指定 Area 发送触发器。如果 Area 已有触发器队列则追加，否则创建新队列 |
| `start_timeout(delay)` | 创建一次性定时器（delay 秒后触发）。递增 timer_id，推入 `CxOsOp::StartTimer { repeats: false }` |
| `start_interval(interval)` | 创建重复定时器（每 interval 秒触发一次）。推入 `CxOsOp::StartTimer { repeats: true }` |
| `stop_timer(timer)` | 停止定时器。推入 `CxOsOp::StopTimer` |
| `event_id()` | 返回当前事件 ID |

### 全局单例存储

| 方法 | 说明 |
|------|------|
| `set_global::<T>(value)` | 用 TypeId 为键存储全局值。检查是否已存在，不存在时插入 |
| `get_global::<T>()` | 按 TypeId 获取全局可变引用。查找不到会 panic |
| `has_global::<T>()` | 检查是否存在指定类型的全局值 |
| `global::<T: Default>()` | 获取或创建全局值（不存在时创建默认值） |

### 网络

| 方法 | 说明 |
|------|------|
| `spawner()` | 返回 futures spawner 引用 |
| `http_request(request_id, request)` | 通过 net 运行时发起 HTTP 请求 |
| `cancel_http_request(request_id)` | 取消 HTTP 请求 |
| `set_thread_priority(priority)` | 设置当前线程优先级（仅在 Android 上有效） |
| `get_ref()` | 获取 CxRef 克隆 |

### 光标/滚动/拖放

| 方法 | 说明 |
|------|------|
| `set_cursor(cursor)` | 设置鼠标光标。存在已有的 SetCursor 操作则替换，否则追加 |
| `sweep_lock(value)` / `sweep_unlock(value)` | 锁定/解锁手指滑扫（在手指系统中注册 Area） |
| `is_scrolling_allowed_within(area)` | 判断指定 area 内是否允许滚动。检查 `blocked_scrolling_exception_area()` |
| `block_scrolling_except_within(scrollable_area)` | 全局阻止滚动，仅允许在指定 area 内滚动 |
| `unblock_scrolling()` | 恢复所有区域允许滚动 |
| `start_dragging(items)` | 启动拖放操作。检查没有重复的拖放操作 |
| `push_unique_platform_op(op)` | 向 platform_ops 推送不重复的操作（去重） |

### 剪贴板/选择/无障碍

| 方法 | 说明 |
|------|------|
| `show_clipboard_actions(has_selection, rect, keyboard_shift)` | 显示原生剪贴板操作菜单。在 Android 上使用 ActionMode，iOS 使用 UIMenuController |
| `hide_clipboard_actions()` | 隐藏剪贴板菜单 |
| `copy_to_clipboard(content)` | 复制文本到剪贴板（Web/tvOS 不支持） |
| `set_primary_selection(content)` | 设置主选择缓冲区（Linux 中键粘贴） |
| `update_accessibility_tree(update)` | 推送无障碍树更新。update 是类型擦除的 `accesskit::TreeUpdate` |
| `show_selection_handles(start, end)` / `update_selection_handles(start, end)` / `hide_selection_handles()` | 移动端选择手柄控制 |

### macOS 菜单 / Dock / 系统栏

| 方法 | 说明 |
|------|------|
| `update_macos_menu(menu)` | 更新 macOS 菜单栏 |
| `show_in_dock(show)` | 控制是否在 dock 中显示应用图标 |
| `set_system_bar_appearance(appearance)` | 设置系统栏图标颜色（Android）。支持 `Auto`（自动根据背景亮度选择）、`DarkIcons`、`LightIcons` |

### 系统浏览器

| 方法 | 说明 |
|------|------|
| `browser_update_url(url, replace)` | 更新浏览器 URL（委托到 CxOsApi） |
| `browser_history_go(delta)` | 浏览器历史导航（委托到 CxOsApi） |
| `system_browser(id)` | 创建 `CxSystemBrowser` 操作句柄 |

### 退出

| 方法 | 说明 |
|------|------|
| `quit()` | 推送 `CxOsOp::Quit` 退出应用 |
| `request_quit(reason)` | 请求退出。先向事件处理器发送 `Event::QuitRequested`，如果未被处理则调用 `quit()`。返回 `handled` 状态 |

### 权限

| 方法 | 说明 |
|------|------|
| `request_permission(permission)` | 请求权限，推入 `CxOsOp::RequestPermission`，返回 request_id |
| `check_permission(permission)` | 检查权限状态，返回 request_id |

### 视频播放

| 方法 | 说明 |
|------|------|
| `prepare_video_playback(...)` | 准备视频播放（默认使用 Camera 权限）。委托给 `prepare_video_playback_with_permission` |
| `prepare_headset_camera_playback(...)` | 准备头戴相机播放（使用 HeadsetCamera 权限） |
| `prepare_video_playback_with_permission(...)` | 内部方法。如果 source 是 Camera，检查权限：将待处理的播放推入 `pending_camera_playbacks` 后请求权限；权限已有时直接推入 `CxOsOp::PrepareVideoPlayback` |
| `handle_camera_permission_result(result)` | 处理相机权限结果。如果是 Camera/HeadsetCamera 权限，遍历 `pending_camera_playbacks`：已授权则执行 `PrepareVideoPlayback`；被拒绝则发送 `Event::VideoDecodingError` |
| `attach_camera_native_preview(video_id, area)` | 附着相机原生预览 |
| `update_camera_native_preview(video_id, area, visible)` | 更新相机原生预览 |
| `detach_camera_native_preview(video_id)` | 分离相机原生预览 |
| `begin_video_playback(video_id)` | 开始视频播放 |
| `pause_video_playback(video_id)` | 暂停视频播放 |
| `resume_video_playback(video_id)` | 恢复视频播放 |
| `mute_video_playback(video_id)` | 静音 |
| `unmute_video_playback(video_id)` | 取消静音 |
| `cleanup_video_playback_resources(video_id)` | 清理视频资源 |
| `cancel_pending_camera_playback(video_id)` | 取消待处理的相机播放请求 |
| `seek_video_playback(video_id, position_ms)` | 跳转到指定时间 |
| `set_video_volume(video_id, volume)` | 设置音量 |
| `set_video_playback_rate(video_id, rate)` | 设置播放速度 |

### 音频播放

| 方法 | 说明 |
|------|------|
| `prepare_audio_playback(video_id, source, autoplay, should_loop)` | 准备音频播放。通过 `CxOsOp::PrepareAudioPlayback` 实现 |

### 文件对话框

| 方法 | 说明 |
|------|------|
| `open_system_savefile_dialog()` | 打开保存文件对话框 |
| `open_system_openfile_dialog()` | 打开选择文件对话框 |
| `open_system_savefolder_dialog()` | 打开保存文件夹对话框 |
| `open_system_openfolder_dialog()` | 打开选择文件夹对话框 |

### 诊断

| 方法 | 说明 |
|------|------|
| `println_resources()` | 打印纹理数量到控制台 |

### XR 方法

| 方法 | 说明 |
|------|------|
| `xr_start_presenting()` | 推送 XrStartPresenting |
| `xr_set_render_scale(scale)` | 推送 XrSetRenderScale |
| `xr_advertise_anchor(anchor)` | 推送 XrAdvertiseAnchor |
| `xr_set_local_anchor(anchor)` | 推送 XrSetLocalAnchor |
| `xr_set_local_floor(floor_y)` | 推送 XrSetLocalFloor |
| `xr_discover_anchor(id)` | 推送 XrDiscoverAnchor |

### DPI 覆盖辅助

| 方法 | 说明 |
|------|------|
| `dpi_override_scale(pos, window_id)` | 将 OS 报告的坐标从逻辑点重映射到布局逻辑点（考虑 DPI 覆盖）。平台事件处理器在触控/滚轮事件 abs 字段上调用此方法 |

---

## 自由函数

### `can_play_type(mime)`（第 1698-1743 行）

返回指定 MIME 类型在当前平台上的可播放性：`""`（不能播放）、`"maybe"`、`"probably"`。根据编译目标平台（Linux/Android/macOS/iOS/tvOS/Windows）委托到各平台的视频播放器实现。

---

## 宏

### `register_component_factory!`（第 1745-1781 行）

用于在运行时组件注册表中注册组件工厂。通过 `module_path!()` 获取模块 ID，检查是否冲突后插入 `LiveType -> (LiveComponentInfo, Box<factory>)` 映射。
