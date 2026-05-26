# mod.rs — Apple 平台模块入口

**文件路径:** `platform/src/os/apple/mod.rs`

**核心目的:** 作为 Makepad 框架中 Apple 平台（macOS/iOS/tvOS）支持的模块根文件。它声明了各个子模块并引入必要的依赖项，是平台特定代码的入口点。

**模块声明:**
- `mod apple_classes` — Apple Objective-C 运行时类桥接
- `mod apple_sys` — 重新导出原始 FFI 绑定 (`makepad_apple_sys`)
- `mod apple_util` — 跨平台实用工具（光标、字符串转换、事件）
- `mod apple_game_input` — 游戏控制器输入处理
- `mod apple_media` — 媒体元数据（iTunes/Music.app）
- `mod apple_resources` — bundle 资源加载
- `mod apple_video_playback` — 视频播放流水线
- `mod apple_video_player` — 视频播放器 UI 控件
- `mod apple_webview` — WebView 集成
- `mod apple_yuv_metal` — YUV→RGB 着色器转换
- `mod audio_tap` — 系统音频捕获
- `mod audio_unit` — CoreAudio AudioUnit 包装
- `mod av_capture` — 摄像头捕获（AVCapture）
- `mod core_midi` — CoreMIDI 输入
- `mod metal` — Metal GPU 渲染后端

**平台条件编译:**
- `mod tvos` — tvOS 平台 (target_os = "tvos")
- `mod macos` — macOS 平台 (target_os = "macos")

**依赖:** `std::sync::OnceLock`, `makepad_platform::*`, `makepad_apple_sys`
