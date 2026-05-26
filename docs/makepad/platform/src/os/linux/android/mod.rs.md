# Android 平台模块声明

## 概述

`mod.rs` 是 Android 平台子系统的模块根文件。它声明了该目录下所有公共子模块，构成 Makepad Android 平台实现的基础骨架。

## 模块列表

| 模块名 | 用途 |
|--------|------|
| `aaudio_sys` | AAudio 音频 API 的 FFI 绑定（常量、类型、外部函数声明） |
| `acamera_sys` | Android Camera2 NDK API 的 FFI 绑定 |
| `amidi_sys` | Android MIDI (AMidi) API 的 FFI 绑定（运行时动态加载） |
| `android` | Android 主平台实现：窗口管理、事件循环、OpenGL/XR 渲染、视频播放、网络、IME、剪贴板、权限等 |
| `android_audio` | 基于 AAudio 的音频输入/输出实现 |
| `android_camera` | 基于 Camera2 NDK 的摄像头预览与视频编码实现 |
| `android_camera_player` | 将 Android NDK 摄像头作为视频播放源使用 |
| `android_jni` | JNI 桥接层：从 Java 接收事件的 extern "C" 函数、发往 Java 的辅助函数 |
| `android_keycodes` | Android 键码到 Makepad KeyCode 的映射 |
| `android_media` | 媒体子系统聚合：管理音频、MIDI、摄像头的懒初始化和信号分发 |
| `android_midi` | 基于 AMidi 的 MIDI 输入/输出实现 |
| `android_network` | Android 平台网络后端：HTTP 请求和 WebSocket（通过 Java 桥接） |
| `android_video_playback` | 视频播放配置与能力检测 |
| `ndk_sys` | Android NDK 核心类型和 FFI 绑定（ANativeWindow、AAssetManager、AChoreographer、ANativeActivity） |
| `ndk_utils` | JNI 辅助宏（call_method、new_object、字符串/引用操作） |

## 实现说明

- 所有模块均通过 `pub mod` 公开导出，供上层 `super::android` 或其他平台集成代码使用。
- `android.rs` 是最大最核心的模块（约 3455 行），包含完整的 Android 平台事件循环。
- `ndk_sys`、`aaudio_sys`、`acamera_sys`、`amidi_sys`、`drm_sys`、`gbm_sys` 是 FFI 绑定模块，只声明外部函数和类型。
- `ndk_utils` 是纯宏模块，不包含运行时逻辑。
