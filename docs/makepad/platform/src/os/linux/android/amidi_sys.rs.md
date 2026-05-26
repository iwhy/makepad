# Android MIDI (AMidi) FFI 绑定

## 概述

`amidi_sys.rs` 是 Android AMidi API 的运行时动态加载 FFI 绑定。`libamidi.so` 从 Android 10 (API 29) 开始可用。为了保持对更低 API 级别（minSdk 26）的兼容性，所有符号通过 `dlopen` + `dlsym` 在运行时解析。

当 `libamidi.so` 不可用（API < 29）时，所有函数优雅地返回错误码，不崩溃。

## 类型

| 类型 | 说明 |
|------|------|
| `AMidiDevice` | MIDI 设备（不透明） |
| `AMidiInputPort` | MIDI 输入端口（不透明） |
| `AMidiOutputPort` | MIDI 输出端口（不透明） |
| `media_status_t` | 媒体状态码（`c_int`） |
| `AMidiVtable` | 存储 dlsym 解析出的函数指针 |

## 运行时加载机制

```rust
static AMIDI: OnceLock<Option<AMidiVtable>> = OnceLock::new();
```

- `AMIDI` 是 `OnceLock`，确保只加载一次 `libamidi.so`。
- 使用 `ModuleLoader::load("libamidi.so")` 加载库。
- 使用 `ModuleLoader::get_symbol()` 解析每个符号。
- 加载成功后，库句柄通过 `std::mem::forget` 泄露以保持符号有效。
- 加载失败时缓存 `None`，整个进程生命周期内不再重试。

## 函数（按 AMidi 原命名）

| 函数 | 说明 |
|------|------|
| `AMidiDevice_fromJava` | 从 Java `MidiDevice` 对象创建 NDK 设备 |
| `AMidiDevice_release` | 释放设备 |
| `AMidiDevice_getNumInputPorts` | 获取输入端口数 |
| `AMidiDevice_getNumOutputPorts` | 获取输出端口数 |
| `AMidiOutputPort_open` | 打开输出端口（接收数据） |
| `AMidiOutputPort_close` | 关闭输出端口 |
| `AMidiOutputPort_receive` | 接收 MIDI 数据 |
| `AMidiInputPort_open` | 打开输入端口（发送数据） |
| `AMidiInputPort_send` | 发送 MIDI 数据 |
| `AMidiInputPort_close` | 关闭输入端口 |

## 错误处理

- `AMIDI_UNAVAILABLE = -1` — 库不可用时返回
- 当 `AMidiDevice_getNumInputPorts` 失败时返回 `0`（安全默认值）
- `AMidiOutputPort_receive` 失败时返回 `0`（表示没有消息）
- `AMidiInputPort_send` 失败时返回 `-1`

## 实现说明

- 函数指针类型（`Fn_AMidiDevice_fromJava` 等）匹配原始 `extern "C"` 签名。
- `AMidiVtable` 手动实现了 `Send` 和 `Sync`（所有字段为原始函数指针，线程安全）。
- 实际使用实现在 `android_midi.rs` 中，通过 `AMidiOutputPort_open`/`AMidiInputPort_open` 等打开具体端口。
- 文档注释中提到的 `to_java.open_all_midi_devices()` 等 JNI 调用在 `android.rs` 中处理。
