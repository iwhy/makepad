# apple_resources.rs — Apple Bundle 资源加载

**文件路径:** `platform/src/os/apple/apple_resources.rs`

**核心目的:** 提供从 Apple 应用程序 bundle 中加载资源文件的函数，支持 macOS 的 `.app` bundle 结构和 iOS/tvOS 的扁平 bundle 结构。

**关键函数:**
| 函数 | 描述 |
|------|------|
| `resource_dir_path()` | 获取资源目录路径。macOS 返回 `bundle_path/Contents/Resources`；iOS/tvOS 直接返回 `bundle_path` |
| `bundle_path()` | 获取主 bundle 目录路径 |
| `load_self_resource(path)` | 从 bundle 的 Resources 目录加载资源文件，返回 `Option<Vec<u8>>` |

**实现细节:**
- 在 macOS 上通过 `_NSBundle_mainBundle` 和 `_NSBundle_bundlePath` FFI 调用获取 bundle 路径
- 在 iOS/tvOS 上同样使用 `_NSBundle_mainBundle`，但 bundle 结构扁平，不需要 `Contents/Resources` 后缀
- 使用全局 `FILE_CACHE` (`OnceLock<HashMap>`) 缓存已加载资源的文件路径
- `load_self_resource` 会优先检查内存缓存以加速重复加载

**平台集成:** macOS 和 iOS/tvOS 之间资源目录结构差异在此抽象
