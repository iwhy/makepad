# Android NDK 核心 FFI 绑定

## 概述

`ndk_sys.rs` 是 Android NDK 核心类型的原始 FFI 绑定。涵盖了 `libandroid.so` 中的基本类型和函数：ANativeWindow、AHardwareBuffer、AAssetManager、AChoreographer、ANativeActivity 等。

## 不透明类型

| 类型 | 说明 |
|------|------|
| `ANativeWindow` | 原生窗口（Surface） |
| `AHardwareBuffer` | 硬件缓冲区（API 26+） |
| `AAssetManager` | 资源管理器 |
| `AAsset` | 资源文件句柄 |

## 具象类型

| 类型 | 说明 |
|------|------|
| `AChoreographer` | 帧同步器（`c_void` 别名） |
| `AChoreographerFrameCallbackData` | 帧回调数据（`c_void` 别名） |
| `AChoreographer_vsyncCallback` | Vsync 回调函数类型 |
| `AChoreographerPostCallbackFn` | 通用 post-callback 函数类型 |
| `ANativeActivity` | 原生 Activity（含 callbacks、VM、sdkVersion、assetManager 等） |
| `ANativeActivityCallbacks` | Activity 生命周期回调表 |
| `AInputQueue` | 输入队列 |
| `ARect` | 矩形（left/top/right/bottom） |

## 外部函数

通过 `#[link(name = "android")]` 链接 `libandroid.so`：

### 资源管理
- `AAssetManager_open` — 打开资源文件
- `AAsset_getLength64` — 获取资源长度
- `AAsset_read` — 读取资源数据
- `AAsset_close` — 关闭资源
- `AAssetManager_fromJava` — 从 Java AssetManager 对象获取 NDK 句柄

### 窗口管理
- `ANativeWindow_fromSurface` — 从 Java Surface 创建 ANativeWindow
- `ANativeWindow_release` — 释放窗口引用

### 硬件缓冲区
- `AHardwareBuffer_acquire` / `AHardwareBuffer_release` — 引用计数管理

### 帧同步
- `AChoreographer_getInstance` — 获取全局 Choreographer 实例（API 24+，无条件链接安全）

## 常量

- `AASSET_MODE_BUFFER = 3` — 资源读取模式（缓冲区模式）
- `AHARDWAREBUFFER_USAGE_CPU_READ_RARELY = 2`
- `AHARDWAREBUFFER_USAGE_CPU_READ_OFTEN = 3`
- `AHARDWAREBUFFER_USAGE_GPU_SAMPLED_IMAGE = 1 << 8`

## 设计决策

### dlsym 策略（非 extern "C" 静态链接）

以下函数 **故意不用** `extern "C"` 声明，而是通过 `dlsym` 在运行时解析：

- `ANativeWindow_setFrameRate` — API 30+；API 26-29 设备上没有此符号
- `AChoreographer_postVsyncCallback` — API 33+
- `AChoreographer_postFrameCallback64` — API 29+

原因：Rust 的 `extern "C"` 对未定义符号产生强引用，在 API 26-28 设备上会导致 `dlopen` 失败（`UnsatisfiedLinkError`），即使调用处有运行时版本检查。

运行时解析实现在 `android_jni.rs` 的 `initChoreographer` 函数中，使用 `ModuleLoader::load("libandroid.so")` + `dlsym`。

### 回调函数指针

`ANativeActivityCallbacks` 记录了 Android Activity 的完整生命周期回调：
- `onStart` / `onResume` / `onPause` / `onStop` / `onDestroy`
- `onSaveInstanceState`
- `onWindowFocusChanged`
- `onNativeWindowCreated` / `onNativeWindowResized` / `onNativeWindowRedrawNeeded` / `onNativeWindowDestroyed`
- `onInputQueueCreated` / `onInputQueueDestroyed`
- `onContentRectChanged`
- `onConfigurationChanged`
- `onLowMemory`

这些回调在 `android.rs` 中通过 `ANativeActivityCallbacks` 结构体注册，处理 Activity 生命周期事件。
