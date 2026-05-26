# gstreamer_sys.rs — GStreamer FFI 绑定

**文件路径**: `platform/src/os/linux/gstreamer_sys.rs` (229 行)
**核心功能**: GStreamer 多媒体框架的 FFI 绑定，通过 `dlopen` 动态加载，如果系统未安装 GStreamer 则优雅降级。

## 链接库

通过 `ModuleLoader` 动态加载以下共享库：
- `libgstreamer-1.0.so.0` — 核心 GStreamer API
- `libgstapp-1.0.so.0` — AppSink 元素 API
- `libgobject-2.0.so.0` — GLib 对象系统
- `libglib-2.0.so.0` — GLib 工具函数
- `libgstgl-1.0.so.0` — GStreamer GL 集成（可选，零拷贝路径）

## 类型定义

### 不透明指针
`GstElement` / `GstBus` / `GstSample` / `GstBuffer` / `GstCaps` / `GstStructure` / `GstMessage` / `GstMemory` — 均为 `c_void` 类型别名。

### 数据结构
- `GError` — GLib 错误（domain, code, message）
- `GstMapInfo` — 缓冲区内存映射信息（memory, flags, data, size）

### 常量
- 状态：`GST_STATE_NULL` / `PAUSED` / `PLAYING`
- 格式：`GST_FORMAT_TIME`
- Seek 标志：`FLUSH` / `ACCURATE` / `KEY_UNIT`
- 消息类型：`GST_MESSAGE_ERROR`

## `LibGStreamer` 结构体

### 核心 GStreamer 函数
`gst_init` / `gst_element_factory_make` / `gst_element_set_state` / `gst_element_get_state` / `gst_element_get_bus` / `gst_bus_pop_filtered` / `gst_message_parse_error`。

### AppSink 函数
`gst_app_sink_try_pull_preroll` / `gst_app_sink_try_pull_sample` / `gst_app_sink_is_eos` / `gst_app_sink_set_caps`。

### Buffer 操作
`gst_sample_get_buffer` / `gst_sample_get_caps` / `gst_buffer_peek_memory` / `gst_buffer_map` / `gst_buffer_unmap`。

### Caps 操作
`gst_caps_from_string` / `gst_caps_unref` / `gst_caps_get_structure` / `gst_structure_get_int`。

### Seek/Query
`gst_element_seek_simple` / `gst_element_seek` / `gst_element_query_position` / `gst_element_query_duration` / `gst_query_new_seeking` / `gst_query_new_buffering`。

### GL 零拷贝（可选）
`gst_is_gl_memory` / `gst_gl_memory_get_texture_id` — 检测和获取 GLMemory 中的纹理 ID。

### GLib 函数
`g_object_set_*`（字符串/整数/指针/浮点数变体）、`g_free`、`g_error_free`。

## 线程安全性

`LibGStreamer` 实现了 `Send + Sync`，设计为单线程使用。
