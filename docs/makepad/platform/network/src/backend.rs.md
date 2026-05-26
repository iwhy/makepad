# backend.rs — 网络后端抽象层

**File path**: `platform/network/src/backend.rs` (190 行)
**Core purpose**: 定义平台无关的网络后端 trait `NetworkBackend`，提供事件发送通道 `EventSink`，以及按目标平台选择默认后端的编译时逻辑。

## 模块声明

条件编译的子模块：
- `android` — `#[cfg(target_os = "android")]`
- `apple` — `#[cfg(any(ios, macos, tvos))]`
- `linux` — `#[cfg(target_os = "linux")]`
- `web` — `#[cfg(target_arch = "wasm32")]`
- `windows` — `#[cfg(target_os = "windows")]`

## EventSink
- `sender: Sender<NetworkResponse>` — mpsc 发送端
- `wake_fn: Arc<Mutex<Option<Arc<dyn Fn() + Send + Sync>>>>` — 可选唤醒函数
- `emit(event)` — 发送事件并调用 wake_fn
- `set_wake_fn(fn)` — 设置/清除唤醒函数

## NetworkBackend trait
```rust
pub trait NetworkBackend: Send + Sync + 'static {
    fn http_start(&self, request_id: LiveId, request: HttpRequest, sink: EventSink) -> Result<(), NetworkError>;
    fn http_cancel(&self, request_id: LiveId) -> Result<(), NetworkError>;
    fn ws_open(&self, socket_id: LiveId, request: HttpRequest, sink: EventSink) -> Result<(), NetworkError>;
    fn ws_send(&self, socket_id: LiveId, message: WsSend) -> Result<(), NetworkError>;
    fn ws_close(&self, socket_id: LiveId) -> Result<(), NetworkError>;
}
```

## UnsupportedBackend
- 占位后端，所有操作返回 `NetworkError::Unsupported(reason)`
- 用于不支持网络的平台

## default_backend()

条件编译选择默认后端：
- WASM → `web::create_backend()`
- Android → `android::create_backend()`
- Windows → `windows::create_backend()`
- Apple → `apple::create_backend()`
- Linux → `linux::create_backend()`
- 其他 → `UnsupportedBackend`
