# oh_callbacks.rs

One-liner (EN): Native callback registration for XComponent surface lifecycle, touch dispatch, VSync signals, and text/keyboard IME events from ArkTS to Rust.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/open_harmony/oh_callbacks.rs` (275 行)
- **核心作用**: 注册 XComponent（原生渲染表面）和 VSync（垂直同步）的原生回调，将触摸事件、文本输入、键盘事件从 ArkTS 层通过 mpsc 通道转发到 Makepad 的 Rust 事件循环。

## 类型/结构体

### `FromOhosMessage` — 从 ArkTS 到 Rust 的消息枚举

| 变体 | 数据 | 说明 |
|------|------|------|
| `Init` | `device_type, os_full_name, display_density, files_dir, cache_dir, temp_dir, raw_env, arkts_ref, raw_file` | 应用初始化参数，由 `ohos_ability_on_create` 触发 |
| `SurfaceCreated` | `window, width, height` | XComponent 渲染表面已创建 |
| `SurfaceChanged` | `window, width, height` | 表面尺寸/配置改变 |
| `SurfaceDestroyed` | — | 表面已销毁 |
| `VSync` | — | 垂直同步信号 |
| `Touch(Vec<TouchPoint>)` | 触摸点列表 | 触摸事件（按下/移动/抬起/取消）|
| `TextInput(TextInputEvent)` | 文本输入事件 | IME 文本提交 |
| `DeleteLeft(i32)` | 删除长度 | IME 向左删除 |
| `ResizeTextIME(bool, i32)` | 键盘状态 + 高度 | IME 键盘显示/隐藏 |

### 辅助结构体

- **`VSyncParams`** — 内部结构，持有 `OH_NativeVSync*` 指针和 mpsc 发送端，在 C 回调中通过 `data` 指针传递

### 线程局部存储

- **`OHOS_MSG_TX`** — `thread_local!` 静态变量，存储 `mpsc::Sender<FromOhosMessage>`，使 `#[napi]` 导出的函数（在 napi 线程调用）能发送消息到主事件循环

## 关键函数

### napi 导出函数（由 ArkTS 调用）

| napi 函数名 | 用途 |
|------------|------|
| `handle_insert_text_event(text: String)` | IME 文本输入 → `FromOhosMessage::TextInput` |
| `handle_delete_left_event(length: i32)` | IME 删除 → `FromOhosMessage::DeleteLeft` |
| `handle_keyboard_status(is_open: bool, keyboard_height: i32)` | IME 键盘状态变化 → `FromOhosMessage::ResizeTextIME` |

### C 回调函数（由 OpenHarmony 框架调用）

| C 函数 | 注册位置 | 触发条件 |
|--------|----------|----------|
| `on_surface_created_cb` | `OH_NativeXComponent_Callback::OnSurfaceCreated` | XComponent 渲染表面创建 |
| `on_surface_changed_cb` | `OnSurfaceChanged` | 表面尺寸/配置变化 |
| `on_surface_destroyed_cb` | `OnSurfaceDestroyed` | 表面销毁 |
| `on_dispatch_touch_event_cb` | `DispatchTouchEvent` | 触摸事件分发 |
| `on_vsync_cb` | `OH_NativeVSync_RequestFrame` | 垂直同步信号（**循环注册**） |
| `on_frame_cb` | 预留 (unused) | XComponent 帧回调 |

### 注册函数

| 函数 | 用途 |
|------|------|
| `init_globals(from_ohos_tx)` | 初始化线程局部存储的 mpsc 发送端 |
| `register_xcomponent_callbacks(env, xcomponent)` | 从 JsObject 中 unwrap `OH_NativeXComponent` 并注册生命周期和触摸回调 |
| `register_vsync_callback(from_ohos_tx)` | 创建 `OH_NativeVSync` 并注册循环 VSync 回调 |
| `send_from_ohos_message(message)` | 通过线程局部存储发送消息到主循环 |

## 实现细节

- **VSync 循环**: `on_vsync_cb` 在处理完信号后立即调用 `OH_NativeVSync_RequestFrame` 注册下一帧回调，形成连续 VSync 流。
- **触摸处理**: 将 `OH_NativeXComponent_TouchEvent` 的原始坐标（像素）、时间戳（纳秒 → 秒）、压力值转换为 Makepad 的 `TouchPoint` 结构体。触控类型映射: `DOWN→Start`, `UP→Stop`, `MOVE→Move`, `CANCEL→Move`。
- **非安全实现**: `FromOhosMessage` 包含原始指针，通过 `unsafe impl Send` 标记以通过通道传输。
- **XComponent 回调注册**: 使用 `Box::leak` 将回调结构体分配到静态内存中（OpenHarmony 要求回调指针在组件生命周期内有效）。
