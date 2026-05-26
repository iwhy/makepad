# `audio.rs` — 音频设备管理与音频缓冲区操作

## 概述
该文件定义了 Makepad 音频子系统的核心数据类型，涵盖音频设备描述、音频缓冲区操作、以及设备事件的查询过滤。它是平台层音频 I/O 的基础类型定义模块，不涉及具体后端实现。

---

## 核心常量

- **`MAX_AUDIO_DEVICE_INDEX`** (32)：支持的最大音频设备索引上限，用于枚举循环的边界保护。

---

## 回调函数类型

- **`AudioOutputFn`**：`Box<dyn FnMut(AudioInfo, &mut AudioBuffer) + Send + 'static>`。音频输出回调，接收当前音频信息和一个可变的输出缓冲区，调用方需填充音频数据到缓冲区中。
- **`AudioInputFn`**：`Box<dyn FnMut(AudioInfo, &AudioBuffer) + Send + 'static>`。音频输入回调，接收当前音频信息和一个只读的输入缓冲区，调用方从中读取采集到的音频数据。

---

## 数据结构

### `AudioDeviceId`
包装 `LiveId` 的音频设备唯一标识符，支持 `Clone`/`Copy`/`Hash`/`FromLiveId` 等特性，方便在事件和配置中传递。

### `AudioInfo`
描述一次音频回调的上下文信息：
- **`device_id`**：产生该回调的设备标识。
- **`time`**：可选的 `AudioTime` 时间戳，用于精确的时序同步和延迟计算。
- **`sample_rate`**：当前的采样率（Hz），输出回调据此计算填充帧数。

### `AudioDeviceDesc`
音频设备的完整描述，包含设备 ID、类型（输入/输出/环回）、是否为默认设备、是否发生过故障、声道数、以及人类可读的名称。`Display` 实现以 `[Default Input]` 或 `[Output]` 等格式输出。

### `AudioDevicesEvent`
当系统音频设备列表发生变化时触发的事件，内部维护一个 `Vec<AudioDeviceDesc>`。
- **`default_input()`**：遍历设备列表，优先返回未故障的默认输入设备 ID；若无可用默认输入则回退到任意默认输入设备；若都找不到则返回空向量。
- **`default_output()`**：与 `default_input()` 逻辑对称，返回默认输出设备 ID，同样优先过滤已故障的设备。
- **`match_outputs(outputs)`**：按设备名称子字符串匹配输出设备，支持传入多个关键词进行模糊匹配；若未匹配到任何设备则回退到 `default_output()`。
- **`match_inputs(inputs)`**：按设备名称子字符串匹配输入设备，与 match_outputs 不同之处在于不提供回退逻辑，匹配不到则返回空向量。

### `AudioDeviceType`
枚举音频设备的工作方向：
- **`Input`**：输入设备（麦克风等）。
- **`Output`**：输出设备（扬声器等）。
- **`Loopback`**：环回设备，本质上是输出设备但以输入方式打开，用于捕获系统播放的音频。

类型判断方法 `is_input()` 将 Loopback 视为输入，`is_output()` 仅判断 Output，`is_loopback()` 仅判断 Loopback。

### `AudioTime`
音频时序信息，包含：
- **`sample_time`**：以采样点数为单位的浮点时间计数。
- **`host_time`**：宿主机纳秒级时间戳，用于与系统时钟关联。
- **`rate_scalar`**：速率缩放因子，用于将 sample_time 转换为主机时间。

### `AudioBuffer`
核心音频 PCM 缓冲区，采用**分离声道平面格式**（非交错，planar），数据以 `Vec<f32>` 存储，取值范围归一化为 `[-1.0, 1.0]`。

**构造方法：**
- **`from_data(data, channel_count)`**：从已有的 `Vec<f32>` 构造缓冲区，自动计算 `frame_count` 为 `data.len() / channel_count`。
- **`from_i16(inp, channel_count)`**：从 i16 整数 PCM 数据构造缓冲区，将每个样本除以 `32767.0` 归一化为 f32。
- **`new_with_size(frame_count, channel_count)`**：分配指定帧数和声道数的空缓冲区。
- **`new_like(like)`**：按已有缓冲区的帧数和声道数创建相同规格的空缓冲区。

**格式转换与访问：**
- **`make_single_channel()`**：将多声道数据缩减为单声道，仅保留第一声道的帧数，其余数据被丢弃。
- **`into_data()`**：消费 `AudioBuffer` 返回内部 `Vec<f32>`。
- **`to_i16()`**：将归一化的 f32 数据转换回 i16 整数 PCM，乘以 `32767.0` 后 clamp 到 `i16` 合法范围。
- **`stereo_mut()` / `stereo()`**：对双声道缓冲区按帧数分半，返回左右声道的可变/不可变切片引用；非双声道时 panic。
- **`channel_mut(channel)` / `channel(channel)`**：返回指定声道的帧数据切片，从第 `channel * frame_count` 偏移处开始。
- **`zero()`**：将缓冲区所有样本置零。
- **`copy_from_interleaved(channel_count, interleaved)`**：从交错格式（interleaved）数据复制到内部平面格式，输入为 `[L0,R0,L1,R1,...]` 排列。
- **`copy_to_interleaved(interleaved)`**：将内部平面格式输出为交错格式，输出为 `[L0,R0,L1,R1,...]` 排列，目标缓冲区长度必须匹配。

**大小管理：**
- **`resize(frame_count, channel_count)`**：调整缓冲区大小，仅在帧数或声道数变化时触发重新分配。若设置了 `final_size` 标志且试图改变大小则 panic。
- **`resize_like(like)`**：按参考缓冲区调整大小。
- **`copy_from(like)`**：先 resize 再逐元素复制数据。
- **`set_final_size()` / `clear_final_size()`**：控制缓冲区是否为最终大小（锁定大小防止意外调整）。
