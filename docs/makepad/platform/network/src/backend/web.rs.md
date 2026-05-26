# backend/web.rs — WASM 平台后端 Shim

**File path**: `platform/network/src/backend/web.rs` (31 行)
**Core purpose**: 提供 WASM 平台的 `NetworkBackend` 注册机制——实际实现在 JavaScript 侧通过 wasm-bindgen 注册。

## 全局 Slot

使用 `OnceLock<Mutex<Option<Arc<dyn NetworkBackend>>>>` 存储：

### register_platform_backend(backend)
注册 WASM 后端（由 `makepad-platform` js shim 调用）

### clear_platform_backend()
清除注册

### create_backend() -> Arc<dyn NetworkBackend>
- 返回已注册的后端，或 `UnsupportedBackend`（显示 "wasm backend shim not registered"）
