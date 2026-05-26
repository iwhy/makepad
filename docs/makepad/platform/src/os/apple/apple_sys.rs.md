# apple_sys.rs — Apple 原始 FFI 绑定导出

**文件路径:** `platform/src/os/apple/apple_sys.rs`

**核心目的:** 将 `makepad_apple_sys` crate 的全部内容重新导出到 Makepad 平台层。该文件仅一行代码：

```rust
pub use makepad_apple_sys::*;
```

**实现细节:**
- `makepad_apple_sys` 是自动生成的 FFI 绑定 crate，使用 `rust-bindgen` 从 Apple 的 Objective-C 运行时和框架头文件生成
- 包含 CoreFoundation、CoreGraphics、CoreText、CoreVideo、CoreAudio、AVFoundation、Metal、MetalKit、WebKit、GameController、CoreMIDI 等框架的绑定
- 作为 Makepad 平台代码与 Apple 原生 API 之间的薄 FFI 层

**平台集成:** macOS、iOS、tvOS 共享同一组 FFI 绑定，差异通过运行时特性检测处理
