# web_audio.rs — Web Audio API 集成

**文件路径**: `/home/ubuntu/_github/makepad/platform/src/os/web/web_audio.rs` (180 行)
**核心作用**: Web 平台音频输入/输出管理，通过 `WebAudioAccess` 管理音频设备枚举和音频回调，通过 WASM FFI 与 JavaScript Web Audio API 桥接。

## 关键数据结构

### `WebAudioOutputClosure` — 音频输出回调闭包

```rust
#[repr(C)]
pub struct WebAudioOutputClosure {
    pub callback: Box<dyn FnMut(AudioInfo, &mut AudioBuffer) + Send + 'static>,
    pub device_id: AudioDeviceId,
    pub output_buffer: AudioBuffer,
}
```
用于传递给 JS 侧的原生音频输出回调。

### `WebAudioDevice` — Web 音频设备描述

| 字段 | 类型 | 说明 |
|------|------|------|
| `web_device_id` | `String` | JS 侧 Web Audio 设备 ID |
| `desc` | `AudioDeviceDesc` | Makepad 音频设备描述 |

### `WebAudioAccess` — Web 音频访问状态（线程安全）

| 字段 | 类型 | 说明 |
|------|------|------|
| `audio_input_cb` | `[Arc<Mutex<Option<AudioInputFn>>>; MAX_AUDIO_DEVICE_INDEX]` | 音频输入回调数组 |
| `audio_output_cb` | `[Arc<Mutex<Option<AudioOutputFn>>>; MAX_AUDIO_DEVICE_INDEX]` | 音频输出回调数组 |
| `devices` | `Vec<WebAudioDevice>` | 可用音频设备列表 |
| `change_signal` | `SignalToUI` | 设备列表变化信号 |
| `self_arc` | `*const Mutex<WebAudioAccess>` | 自身 Arc 引用（用于 FFI 传参） |
| `output_device_id` | `AudioDeviceId` | 当前输出设备 ID |
| `output_buffer` | `Option<AudioBuffer>` | 输出音频缓冲区 |

## 核心方法

### `WebAudioAccess::new`

初始化时发送 `FromWasmQueryAudioDevices` 查询设备列表，创建自引用 Arc。

### `to_wasm_audio_device_list`

处理 JS 返回的音频设备列表：
1. 清空并重建 `devices` 列表
2. 为每个设备创建 `AudioDeviceDesc`（设备类型 Input/Output，通道数固定 2）
3. 自动标记默认输入/输出设备（优先匹配 web_device_id 为 "default"，否则取第一个）

### `use_audio_inputs`

**当前未实现** — 打印 TODO 日志。

### `use_audio_outputs`

激活音频输出：
- 空设备列表 → 发送 `FromWasmStopAudioOutput` 停止输出
- 多个设备 → 日志提示仅支持单设备
- 发送 `FromWasmStartAudioOutput` 启动输出（含 web_device_id 和上下文指针）

### `get_updated_descs`

返回当前设备列表的 `AudioDeviceDesc` 副本。

## WASM 导出函数

### `wasm_audio_output_entrypoint`

```rust
pub unsafe extern "C" fn wasm_audio_output_entrypoint(context_ptr, frames, channels) -> u32
```

由 JavaScript 调用的音频输出回调：
1. 获取 `WebAudioAccess` 的锁
2. 取出输出缓冲区，调整大小（frames × channels）
3. 调用注册的输出回调函数
4. 返回音频数据指针（`data.as_ptr()`）

**约定**：采样率固定为 48000 Hz。
