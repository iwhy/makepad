# `os/mod.rs` — 平台层 OS 后端选择器

## 文件定位

此文件是 Makepad 平台抽象层的最顶层入口，职责是利用 Rust `#[cfg]` 条件编译属性，根据编译目标（操作系统、架构、feature flag）选择并导出对应的 OS 后端模块。所有上层代码通过 `use crate::os::*` 统一引入，无需关心底层是哪个平台。

---

## 条件编译分支详解

### 1. `cx_native` 模块 — 原生平台 Cx 依赖加载

```rust
#[macro_use]
#[cfg(any(
    target_os = "android", target_os = "linux", target_os = "macos",
    target_os = "ios", target_os = "tvos", target_os = "windows"
))]
pub mod cx_native;
```

- **条件**：所有非 WebAssembly 的原生平台（Android、Linux、macOS、iOS、tvOS、Windows）。
- **作用**：导出 `cx_native.rs` 中实现的 `Cx::native_load_dependencies()` 和 `Cx::time_now()` 方法，这两个方法依赖文件系统和系统时钟 API，在 Web 上不可用。
- `#[macro_use]` 表示该模块可能导出宏。

### 2. `cx_shared` 模块 — 跨平台共享 Cx 方法

```rust
#[macro_use]
pub mod cx_shared;
```

- **无条件导出**：`cx_shared.rs` 中的方法在所有平台（包括 Web）上均可使用。这些方法不依赖特定的 OS 系统调用，只使用 Rust 标准库和内部数据结构。

### 3. `shared_framebuf` 模块 — 共享帧缓冲协议

```rust
pub mod shared_framebuf;
```

- **无条件导出**：`shared_framebuf.rs` 实现了 Studio 远程显示的共享帧缓冲协议。虽然内部包含平台特定代码（通过条件编译区分），但模块本身在所有平台上都可用。

### 4. `termination_signal` 模块 — 进程终止信号处理

```rust
#[cfg(any(target_os = "linux", target_os = "macos", target_os = "windows"))]
pub(crate) mod termination_signal;
```

- **条件**：Linux、macOS、Windows 桌面平台。
- `pub(crate)`：仅 crate 内部可见，不对外开放。
- **作用**：注册 Ctrl+C 信号处理器，实现优雅退出。

### 5. `headless` 模块 — 无头模式

```rust
#[cfg(headless)]
pub mod headless;
#[cfg(headless)]
pub use crate::os::headless::*;
```

- **条件**：启用了 `headless` feature flag。
- 当 `headless` 开启时，整个 OS 层退化为无头实现（没有窗口、没有图形上下文），使用 `use *` 将所有符号提升到 `os` 模块根级别，使上层代码无需区分 headless 与否。

### 6. `apple` 模块 — Apple 平台

```rust
#[cfg(all(
    not(headless),
    any(target_os = "macos", target_os = "ios", target_os = "tvos")
))]
pub mod apple;
pub use crate::os::apple::*;
pub use crate::os::apple::apple_media::*;
```

- **条件**：非 headless 模式下的 macOS、iOS、tvOS。
- 分两步导出：先导出 `apple` 模块的全部公共符号，再专门导出 `apple_media` 模块（处理 Apple 平台的系统媒体能力）。
- `not(headless)` 确保 headless 模式下不会尝试初始化图形栈。

### 7. `windows` 模块 — Windows 平台

```rust
#[cfg(all(not(headless), target_os = "windows"))]
pub mod windows;
#[cfg(all(not(headless), target_os = "windows"))]
pub use crate::os::windows::*;
```

- **条件**：非 headless 模式下的 Windows。
- 注释掉了 `windows_media` 的导出，表明 Windows 平台的媒体子系统尚未完全就绪或已被废弃。

### 8. `linux` 模块 — Linux 平台（含 Android）

```rust
#[cfg(all(not(headless), any(target_os = "android", target_os = "linux")))]
pub mod linux;
#[cfg(all(not(headless), any(target_os = "android", target_os = "linux")))]
pub use crate::os::linux::*;
```

- **条件**：非 headless 模式下的 Android 或 Linux。
- 使用 `pub use *` 将 Linux 后端的全部公共符号提升到模块根。

### 9. `linux_test_stub` — Mac 测试时的 Linux 存根

```rust
#[cfg(all(test, not(headless), target_os = "macos"))]
pub mod linux_test_stub;
#[cfg(all(test, not(headless), target_os = "macos"))]
pub use crate::os::linux_test_stub as linux;
```

- **条件**：仅在 macOS 上运行测试且非 headless 模式时生效。
- 将 `linux_test_stub` 模块重命名为 `linux` 导出，使在 macOS 上运行涉及 Linux 特定路径的测试时可以编译通过。这是一个测试兼容性 hack。

### 10. Android / Linux / OpenHarmony 媒体导出

```rust
#[cfg(all(not(headless), target_os = "android"))]
pub use crate::os::linux::android::android_media::*;

#[cfg(all(not(headless), target_os = "linux", not(target_env = "ohos")))]
pub use crate::os::linux::linux_media::*;

#[cfg(all(not(headless), target_env = "ohos"))]
pub use crate::os::linux::open_harmony::oh_media::*;
```

- 根据具体环境导出对应的媒体子系统实现：
  - **Android**：`android_media`（CAMERA、MediaCodec 等）
  - **标准 Linux**：`linux_media`（基于 V4L2/PipeWire 等）
  - **OpenHarmony**：`oh_media`（OpenHarmony 原生媒体 API）

### 11. `web` 模块 — WebAssembly

```rust
#[cfg(all(not(headless), target_arch = "wasm32"))]
pub mod web;
#[cfg(all(not(headless), target_arch = "wasm32"))]
pub use crate::os::web::*;
```

- **条件**：非 headless 模式下的 wasm32 架构。
- Web 后端使用 DOM / Canvas / WebGL / WebGPU 等浏览器 API。

---

## 模块体系总览

| 编译条件 | 导出的模块 | 作用 |
|---------|-----------|------|
| 任何原生 OS | `cx_native` | 文件加载、系统时钟 |
| 总是 | `cx_shared` | 跨平台共享 Cx 方法 |
| 总是 | `shared_framebuf` | Studio 远程帧缓冲 |
| Linux/macOS/Windows | `termination_signal` | Ctrl+C 信号处理 |
| `headless` | `headless` | 无头模式替代所有图形后端 |
| macOS/iOS/tvOS (!headless) | `apple` + `apple_media` | Apple 原生窗口 + 媒体 |
| Windows (!headless) | `windows` | Windows 原生窗口 |
| Linux/Android (!headless) | `linux` | Linux 原生窗口 |
| test + macOS | `linux_test_stub as linux` | 测试时的 Linux 路径存根 |
| wasm32 (!headless) | `web` | Web 浏览器后端 |

---

## 设计意图

1. **编译期多态**：Makepad 没有使用 trait 对象或动态分发来实现跨平台，而是在编译期通过 `#[cfg]` 直接选择不同的模块编译。这消除了虚函数调用开销，也让 Dead Code Elimination 可以彻底去除未使用的平台代码。

2. **符号统一**：无论哪个平台，最终都通过 `pub use crate::os::*` 导出一致的符号集合（例如 `Cx`, `Window`, `App` 等），上层应用代码无需条件编译。

3. **模块隔离**：每个平台后端是一个独立的 Rust 模块，拥有自己的窗口管理、事件循环、GPU 上下文初始化等实现，互不干扰。

4. **测试兼容**：`linux_test_stub` 的存在说明 Makepad 需要在 macOS 开发机上运行涉及 Linux 路径的测试，通过提供最小存根使条件编译的测试模块可以正常编译。

5. **分层导出**：先 `mod` 声明模块，再 `pub use` 导出符号，且在 Apple 平台上额外导出了 `apple_media` 子模块，体现出对能力的最小化暴露原则。
