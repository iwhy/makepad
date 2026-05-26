# pulse_sys.rs

One-liner (EN): FFI type definitions and function declarations for the PulseAudio audio server API.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/pulse_sys.rs` (529 行)
- **核心作用**: 提供 PulseAudio 音频服务器客户端库的 Rust FFI 绑定，包括上下文管理、流（playback/record）控制、设备枚举和线程安全主循环操作。

## 关键类型别名

| 别名 | 底层类型 | 说明 |
|------|----------|------|
| `pa_io_event_flags_t` | `c_uint` | I/O 事件标志 |
| `pa_context_flags_t` | `c_uint` | 上下文连接标志 |
| `pa_context_state_t` | `c_uint` | 上下文状态枚举 |
| `pa_source_state_t` | `c_int` | 音频源状态枚举 |
| `pa_sample_format_t` | `c_int` | 采样格式枚举 |
| `pa_channel_map_def_t` | `c_uint` | 通道映射默认值 |
| `pa_channel_position_t` | `c_int` | 通道位置枚举 |
| `pa_volume_t` | `u32` | 音量值 |
| `pa_usec_t` | `u64` | 微秒时间 |
| `pa_sink_flags_t` | `c_uint` | 音频输出标志 |
| `pa_sink_state_t` | `c_int` | 音频输出状态枚举 |
| `pa_source_flags_t` | `c_uint` | 音频源标志 |
| `pa_operation_state_t` | `c_uint` | 操作状态 |
| `pa_subscription_event_type_t` | `c_uint` | 订阅事件类型 |
| `pa_stream_state_t` | `c_uint` | 流状态 |
| `pa_seek_mode_t` | `c_uint` | 查找模式 |
| `pa_stream_flags_t` | `c_uint` | 流标志 |

## 关键状态常量

### 上下文状态 (`pa_context_state`)
`PA_CONTEXT_UNCONNECTED`(0), `CONNECTING`(1), `AUTHORIZING`(2), `SETTING_NAME`(3), `READY`(4), `FAILED`(5), `TERMINATED`(6)

### 流状态 (`pa_stream_state`)
`PA_STREAM_UNCONNECTED`(0), `CREATING`(1), `READY`(2), `FAILED`(3), `TERMINATED`(4)

### 操作状态 (`pa_operation_state`)
`PA_OPERATION_RUNNING`(0), `DONE`(1), `CANCELLED`(2)

### 流标志 (`pa_stream_flags`)
`PA_STREAM_START_CORKED`(1), `INTERPOLATE_TIMING`(2), `AUTO_TIMING_UPDATE`(8), `ADJUST_LATENCY`(8192), `START_UNMUTED`(65536)

## 关键结构体

| 结构体 | 说明 |
|--------|------|
| `pa_context` | PulseAudio 上下文（不透明） |
| `pa_stream` | 音频流（不透明） |
| `pa_operation` | 异步操作（不透明） |
| `pa_threaded_mainloop` | 线程安全主循环（不透明） |
| `pa_mainloop_api` | 主循环 API 表（io_new, time_new, defer_new, quit 等函数指针） |
| `pa_sample_spec` | 采样规格（format, rate, channels） |
| `pa_channel_map` | 通道映射（channels + map[32]） |
| `pa_cvolume` | 通道音量（channels + values[32]） |
| `pa_buffer_attr` | 缓冲区属性（maxlength, tlength, prebuf, minreq, fragsize） |
| `pa_sink_info` | 音频输出设备信息（name, index, sample_spec, volume, ports 等） |
| `pa_source_info` | 音频源设备信息（name, index, sample_spec, volume, ports 等） |
| `pa_server_info` | 服务器信息（user_name, host_name, default_sink_name 等） |
| `pa_format_info` | 格式信息（encoding + proplist） |
| `pa_spawn_api` | 产生回调（prefork, postfork, atfork） |

## 回调类型

| 类型 | 签名 | 用途 |
|------|------|------|
| `pa_context_notify_cb_t` | `fn(*mut pa_context, *mut c_void)` | 上下文状态变化通知 |
| `pa_stream_notify_cb_t` | `fn(*mut pa_stream, *mut c_void)` | 流状态变化通知 |
| `pa_stream_request_cb_t` | `fn(*mut pa_stream, usize, *mut c_void)` | 数据请求（写入/读取） |
| `pa_stream_success_cb_t` | `fn(*mut pa_stream, c_int, *mut c_void)` | 操作完成回调 |
| `pa_sink_info_cb_t` | `fn(*mut pa_context, *const pa_sink_info, c_int, *mut c_void)` | 音频输出信息回执 |
| `pa_source_info_cb_t` | `fn(*mut pa_context, *const pa_source_info, c_int, *mut c_void)` | 音频源信息回执 |
| `pa_context_subscribe_cb_t` | `fn(*mut pa_context, pa_subscription_event_type_t, u32, *mut c_void)` | 订阅事件通知 |
| `pa_server_info_cb_t` | `fn(*mut pa_context, *const pa_server_info, *mut c_void)` | 服务器信息回执 |
| `pa_io_event_cb_t` / `pa_time_event_cb_t` / `pa_defer_event_cb_t` | I/O/定时/延迟事件回调 | 主循环事件处理 |

## FFI 函数

### 上下文管理
| 函数 | 说明 |
|------|------|
| `pa_context_new(mainloop, name)` | 创建上下文 |
| `pa_context_new_with_proplist(mainloop, name, proplist)` | 带属性列表创建上下文 |
| `pa_context_connect(c, server, flags, api)` | 连接到 PulseAudio 服务器 |
| `pa_context_disconnect(c)` | 断开连接 |
| `pa_context_unref(c)` | 减少引用计数 |
| `pa_context_get_state(c)` | 获取上下文状态 |
| `pa_context_set_state_callback(c, cb, userdata)` | 设置状态回调 |

### 设备枚举
| 函数 | 说明 |
|------|------|
| `pa_context_get_sink_info_list(c, cb, userdata)` | 枚举音频输出设备 |
| `pa_context_get_source_info_list(c, cb, userdata)` | 枚举音频源设备 |
| `pa_context_get_server_info(c, cb, userdata)` | 获取服务器信息 |

### 流管理
| 函数 | 说明 |
|------|------|
| `pa_stream_new(c, name, ss, map)` | 创建音频流 |
| `pa_stream_set_state_callback(s, cb, userdata)` | 设置流状态回调 |
| `pa_stream_get_state(s)` | 获取流状态 |
| `pa_stream_connect_playback(s, dev, attr, flags, volume, sync_stream)` | 连接播放流 |
| `pa_stream_connect_record(s, dev, attr, flags)` | 连接录音流 |
| `pa_stream_disconnect(s)` | 断开流 |
| `pa_stream_unref(s)` | 减少引用计数 |
| `pa_stream_cork(s, b, cb, userdata)` | 暂停/继续流 |
| `pa_stream_set_write_callback(p, cb, userdata)` | 设置写入回调 |
| `pa_stream_set_read_callback(p, cb, userdata)` | 设置读取回调 |
| `pa_stream_begin_write(p, data, nbytes)` | 开始写入 |
| `pa_stream_write(p, data, nbytes, free_cb, offset, seek)` | 写入音频数据 |
| `pa_stream_writable_size(p)` | 获取可写入字节数 |
| `pa_stream_peek(p, data, nbytes)` | 读取音频数据 |
| `pa_stream_drop(p)` | 丢弃已读取的数据 |

### 主循环操作
| 函数 | 说明 |
|------|------|
| `pa_threaded_mainloop_new()` | 创建线程化主循环 |
| `pa_threaded_mainloop_get_api(m)` | 获取主循环 API 表 |
| `pa_threaded_mainloop_start(m)` | 启动主循环 |
| `pa_threaded_mainloop_stop(m)` | 停止主循环 |
| `pa_threaded_mainloop_lock(m)` | 加锁 |
| `pa_threaded_mainloop_unlock(m)` | 解锁 |
| `pa_threaded_mainloop_wait(m)` | 等待信号 |
| `pa_threaded_mainloop_signal(m, wait_for_accept)` | 发送信号 |
| `pa_threaded_mainloop_free(m)` | 释放主循环 |

### 操作管理
| 函数 | 说明 |
|------|------|
| `pa_operation_get_state(o)` | 获取操作状态 |
| `pa_operation_unref(o)` | 减少引用计数 |

## 实现细节
- 所有不透明结构体使用 `#[repr(C)]` + `_unused: [u8; 0]` 零大小标记类型
- 回调类型均为 `Option<unsafe extern "C" fn(...)>` 以支持空安全
- `pa_mainloop_api` 包含完整的函数指针表：`io_new/enable/free/set_destroy`, `time_new/restart/free/set_destroy`, `defer_new/enable/free/set_destroy`, `quit`
- 链接库: `libpulse` (通过 `#[link(name = "pulse")]`)
- 类型命名遵循 PulseAudio C 惯例：`pa_<subsystem>_<name>_t` 风格
