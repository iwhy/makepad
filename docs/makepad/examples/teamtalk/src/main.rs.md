# teamtalk/src/main.rs

演示 Makepad 的音频 I/O 和网络功能 — 一个 LAN（有线）点对点音频聊天应用。

## 整体结构

### 命令行参数（第25-47行）

- `--device=`：指定音频输入设备
- `--vol=`：麦克风音量（0.0-1.0），通过 `linear_to_log_gain` 转换为对数增益

### 音频处理工具（第63-144行）

- `resample(input, from_rate, to_rate)` — 线性插值重采样
- `apply_limiter(buf, gain)` — 快速启动/慢速释放的平滑限制器
- `calculate_peak(buf)` — 计算平均峰值电平
- `apply_fade_in` / `apply_fade_out` — 对数型淡入/淡出

### 网络协议（第165-178行）

`TeamTalkWire` 枚举（`#[derive(SerBin, DeBin)]`）:
- `Silence`：静音包（client_uid, sequence, frame_count）
- `Audio`：音频数据包（client_uid, sequence, channel_count, 16-bit PCM 数据）

### App 结构体（第154-162行）

使用 `#[new]` 声明底层渲染资源（Window, Pass, DrawList）

### 事件处理（第180-250行）

- `handle_startup`：设置 pass clear color，启动网络栈
- `handle_draw_2d`：空绘制（仅清屏，无 UI）
- `handle_audio_devices`：选择音频输入输出设备
- `handle_signal`：占位
- `AppMain::handle_event`：调用 `match_event_with_draw_2d`

### 网络栈（第253-539行）

`start_network_stack` 启动完整的音频采集 → 网络发送 → 网络接收 → 音频播放流水线：

**音频输入线程**（第282-369行）：
1. 从 `AudioStreamSender` 接收麦克风输入
2. 对每个声道应用限制器
3. 检测活跃/静音状态，应用淡入/淡出
4. 序列化成 `TeamTalkWire::Audio` 或 `Silence`
5. 通过 UDP 广播到 `10.0.0.255:41531`

**网络接收线程**（第371-441行）：
1. 接收 UDP 数据包
2. 反序列化为 `TeamTalkWire`
3. 跟踪每个客户端的序列号，检测丢包和乱序
4. 将音频数据送入混音缓冲

**音频回调**：
- `cx.audio_input(0, |info, buffer| ...)`：采集 → 重采样 → 增益 → 发送
- `cx.audio_output(0, |info, buffer| ...)`：混音 → 重采样 → 输出（支持单声道上混到立体声）
- 含定时检测和回调性能监控

## 关键 API

- `AudioStreamSender::create_pair(min_buf, max_buf)` — 音频流通道
- `AudioBuffer` — 音频缓冲区（多声道）
- `cx.audio_input(port, callback)` — 音频输入回调
- `cx.audio_output(port, callback)` — 音频输出回调
- `cx.use_audio_inputs()` / `cx.use_audio_outputs()` — 选择音频设备
- `SerBin` / `DeBin` — 二进制序列化
- `UdpSocket` — UDP 网络收发
