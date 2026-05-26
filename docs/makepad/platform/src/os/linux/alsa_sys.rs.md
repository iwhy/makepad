# alsa_sys.rs — ALSA C FFI 绑定

**文件路径**: `platform/src/os/linux/alsa_sys.rs` (476 行)
**核心功能**: 对 ALSA（Advanced Linux Sound Architecture）库 `libasound` 的原始 FFI 绑定，包括 PCM 音频和 Sequencer MIDI 两部分。

## 链接库

`#[link(name = "asound")]` — 链接系统级 ALSA 共享库。

## PCM 音频类型与常量

### 主要类型别名
- `snd_pcm_t` — PCM 设备句柄（不透明结构体）
- `snd_pcm_hw_params_t` — 硬件参数配置
- `snd_pcm_format_t` — 音频格式枚举
- `snd_pcm_stream_t` — 流方向（播放/捕获）
- `snd_pcm_access_t` — 访问模式（如交错）

### 关键常量
- `SND_PCM_FORMAT_FLOAT_LE` = 14 — 32 位浮点格式
- `SND_PCM_STREAM_PLAYBACK` = 0 / `SND_PCM_STREAM_CAPTURE` = 1
- `SND_PCM_ACCESS_RW_INTERLEAVED` = 3 — 标准交错读写

### 导出的 PCM 函数
- `snd_pcm_open` / `snd_pcm_hw_params_*` 系列 — PCM 设备打开与参数配置
- `snd_pcm_writei` / `snd_pcm_readi` — 交错格式读写
- `snd_pcm_prepare` / `snd_pcm_hw_params_free` — 状态管理

## Sequencer (MIDI) 类型与常量

### 主要类型
- `snd_seq_t` — Sequencer 句柄
- `snd_seq_event_t` — 通用 MIDI 事件结构体（包含 union 类型的事件数据）
- `snd_seq_addr_t` — 客户端+端口地址
- `snd_seq_port_subscribe_t` — 端口订阅信息
- `snd_seq_client_info_t` / `snd_seq_port_info_t` — 客户端/端口信息查询

### MIDI 事件类型常量
- `SND_SEQ_EVENT_NOTEON/NOTEOFF` / `KEYPRESS` / `CONTROLLER` / `PGMCHANGE` / `CHANPRESS` / `PITCHBEND`
- `SND_SEQ_EVENT_PORT_START/EXIT/CHANGE` — 热插拔通知

### 导出的 Sequencer 函数
- `snd_seq_open` / `snd_seq_set_client_name`
- `snd_seq_create_simple_port`
- `snd_seq_subscribe_port` / `snd_seq_unsubscribe_port`
- `snd_seq_event_input` / `snd_seq_event_output_direct`
- `snd_midi_event_new` / `snd_midi_event_encode` / `snd_midi_event_reset_encode`
- `snd_seq_query_next_client` / `snd_seq_query_next_port`
- 各种 `snd_seq_*_info_*` 信息查询函数

## 设备枚举相关
- `snd_card_next` — 遍历声卡
- `snd_device_name_hint` / `snd_device_name_get_hint` — 获取 PCM 设备名称列表
- `snd_strerror` — 错误码转字符串
