# build.rs — Platform build script: bundle metadata, icons, cfg flags, linker config

**File path**: `platform/build.rs` (179 lines)
**Core purpose**: Generates build-time artifacts (Info.plist for macOS, icon embedding code) and configures conditional compilation flags and linker dependencies per target platform.

## Content
- Writes `makepad-platform.path` marker file (current directory) for downstream package discovery
- **macOS**: Auto-detects app name from workspace root directory (5 levels up from `OUT_DIR`), generates `Info.plist` with `CFBundleName`, `CFBundleIdentifier`, and Game Controller support keys; respects `MAKEPAD_BUNDLE_NAME` and `MAKEPAD_BUNDLE_IDENTIFIER` env vars
- **Icons**: Generates `app_icon_gen.rs` with `include_bytes!()` constants for PNG and ICO icons at 6 resolutions; checks env var overrides first, then falls back to `<workspace_root>/resources/<filename>`
- **MAKEPAD cfg flags**: Parses `MAKEPAD` env var (split by `+`/`,`) and emits `rustc-cfg` for: `lines`, `linux_direct`, `no_android_choreographer`, `quest` (implies `use_gles_3` + `use_vulkan`), `apple_bundle`, `ohos_sim`, `headless`, `use_gles_3`, `use_vulkan`
- **Linking**: Links macOS/iOS/tvOS frameworks (`GameController`, `MetalKit`) and Linux library (`xkbcommon`)
- **Sim detection**: Sets `apple_sim` cfg for `aarch64-apple-*-sim` targets
- Emits `rustc-check-cfg` for all custom conditional compilation flags
