# `video.rs` — 视频编码/解码核心类型与颜色转换

## 概述
该文件是 Makepad 视频子系统的核心类型定义模块，统一管理摄像头帧数据、视频编码器配置、视频解码器配置、编解码能力查询、以及像素格式间的颜色空间转换。它作为平台无关的类型层，为所有平台后端提供一致的 API 抽象。

---

## 平面复制与颜色转换

### `copy_strided_plane(src, width, height, dst)`
从带 stride 步长的源平面复制到紧密排列的目标缓冲区。算法分三种路径：
- 若 `pixel_stride == 1` 且 `row_stride == width`，直接整块内存复制（最优路径）。
- 否则逐行逐像素复制：对每一行计算源行偏移，对行内每个像素按 `pixel_stride` 步进读取，写入目标平坦缓冲区。
- 边界保护通过 `min` 和 `saturating_mul` 防止越界。

### `convert_bgra_8888_to_i420(src, width, height, timestamp_ns, matrix, out)`
将 BGRA 8888 格式的帧转换为 I420（YUV420 planar）格式：
1. 宽度高度和源数据长度验证，分配 Y/U/V 三个平面的内存，Y 平面满分辨率、U/V 平面各四分之一分辨率，分别初始化亮度为 16、色度为 128。
2. 内联函数 `rgb_to_yuv(r, g, b)` 使用 BT.601 系数矩阵完成 RGB 到 YUV 的转换（位移运算代替浮点乘法）。
3. Y 平面逐像素转换：读取 BGRA 排列的像素，取 B/G/R 分量计算 Y，写入 Y 平面对应位置。
4. U/V 平面以 2x2 块为单位下采样：对每个色度像素对应的 2x2 块（处理边缘越界），累加 4 个像素的 U/V 值后取平均。

### `convert_rgba_8888_to_i420(src, width, height, timestamp_ns, matrix, out)`
与 `convert_bgra_8888_to_i420` 逻辑相同，但源数据为 RGBA 排列（先 R 后 G 后 B），其余 YUV 转换和下采样逻辑完全一致。

---

## 设备与格式枚举

### `VideoPixelFormat`
摄像头支持的像素格式枚举，包含 `RGB24`、`YUY2`、`NV12`、`YUV420`、`GRAY`、`MJPEG`、`Unsupported(u32)`。

- **`quality_priority()`**：定义格式的质量优先级排序，`RGB24` 最高（6）→ `GRAY`（1），用于自动选择最优格式。
- **`buffer_to_bgra_32(input, width, height, rgba)`**：将 NV12 格式通过 YUV→RGB 转换输出为 BGRA32 数组。对 NV12 按 2 像素对处理，从交错 NV12 数据中提取 Y1/Y2/U/V，使用 BT.601 系数计算 RGB。
- **`buffer_to_rgb_8(input, rgb, in_width, _in_height, left, top, out_width, out_height)`**：NV12 格式局部区域（ROI）到 RGB8 的转换，支持指定输出区域（left/top/out_width/out_height），使用 YUV→RGB 转换公式。

### `CameraFrameLayout`
摄像头帧的平面布局枚举：`I420`（三平面 Y/U/V）、`NV12`（双平面，Y 平面 + UV 交错平面）、`YUY2`（单平面交错）、`Mjpeg`、`Unknown`。

### `CameraColorMatrix`
颜色矩阵标准枚举：`BT709`、`BT601`、`BT2020`、`Unknown`。方法 `as_yuv_uniform()` 将枚举映射为 shader 可用的浮点值（0.0/1.0/2.0/0.0）。

---

## 帧引用与所有权

### `CameraFramePlaneRef<'a>`
单平面只读引用，包含字节切片 `bytes`、`row_stride`（行步长）、`pixel_stride`（像素步长）。

### `CameraFrameRef<'a>`
完整的摄像头帧引用，包含时间戳、宽高、布局、颜色矩阵、平面数（最多 3 个平面）。`empty()` 构造全零空帧。

### `CameraFramePlaneOwned`
拥有所有权的平面数据，`bytes: Vec<u8>`。

### `CameraFrameOwned`
拥有所有权的完整帧：
- **`reset()`**：清零所有字段，回收内存。
- **`copy_from_ref(src)`**：从 `CameraFrameRef` 复制数据，自动处理 stride 解交织；若源平面与目标布局一致且 stride 连续则直接 memcpy，否则逐行复制。
- **`convert_to_i420(src)`**：将任意支持的布局（I420/NV12/YUY2）转换为标准 I420 格式：
  - I420：直接复制三个平面。
  - NV12：复制 Y 平面，将 UV 交错平面分离为 U、V 两个独立平面。
  - YUY2：从 YUYV 交错数据中提取 Y 值生成全分辨率 Y 平面；对 U/V 值先在行内每像素对提取，再垂直 2x1 下采样取平均得到 U/V 平面。
  - 其他布局返回 false。
- **`plane_size(plane_index)`**：根据布局和平面索引计算平面尺寸（宽度、高度）。I420/NV12 的平面 0 为全分辨率，平面 1/2 为四分之一分辨率。

---

## 帧池与环形缓冲区

### `CameraFramePool`
简单的帧回收池：
- **`new(max_free)`**：创建最大缓存 `max_free` 个帧的池。
- **`checkout()`**：从池中弹出一个空闲帧，池为空时返回默认新帧。
- **`publish_latest(frame)`**：发布新帧替换 `latest`，被替换的旧帧回收到池中。
- **`take_latest()`**：取走当前最新帧，使 `latest` 置空。
- **`recycle(frame)`**：重置帧数据后回收入池（不超过 `max_free` 限制）。

### `CameraFrameRing`
多生产者单消费者的最新帧环形缓冲区，适用于摄像头预览路径：
- 内部维护 `slots`——`Vec<Mutex<CameraFrameOwned>>` 的预分配槽位，`next_write_slot` 原子写指针，`latest_seq` 原子序列号。
- **`new(slot_count)`**：创建至少 2 个槽位的环形缓冲区。
- **`publish_i420_copy(frame_ref)`**：将 I420 帧直接复制发布到环中。
- **`publish_i420_converted(frame_ref)`**：将任意格式帧转换为 I420 后发布。
- **`publish_with(frame_ref, write_frame)`**：内部方法，原子地获取写槽位，执行 `write_frame` 回调写入数据，然后更新 `latest_seq`。若槽位被锁则返回 false。
- **`take_latest(last_seen_seq, out)`**：消费者方法，尝试获取最新帧。通过比较 `last_seen_seq` 避免重复读取；双重检查 `latest_seq` 防止读取中间状态；成功后 swap 取出帧数据。

### `CameraFrameLatest`
消费者端便捷封装，内置游标和待处理状态：
- **`prime_pending_from_latest()`**：从环中拉取最新帧到本地，标记 `has_pending`。
- **`take_pending_or_latest()`**：优先返回已缓存的待处理帧，若没有则尝试从环中拉取最新帧。
- **`pending_frame()`**：查看是否有缓存的待处理帧。

---

## 视频编码器配置

### `VideoEncodeError`
编码器错误枚举：`UnsupportedCodec`、`UnsupportedSource`、`CodecUnavailable`、`EncoderNotStarted`、`InvalidTexture`、`UnsupportedTextureFormat`、`InvalidTextureSize`。

### `VideoCodec`
编码标准枚举：`H264`、`H265`、`Av1`、`Vp8`、`Vp9`。

### `VideoBitstreamFormat`
码流格式枚举：`AnnexB`（H.264/H.265 起始码格式）、`Avcc`（H.264 length-prefixed）、`Av1Obu`（AV1 OBU 格式）、`RawAccessUnit`。

### `VideoQueuePolicy`
帧队列策略，当前仅 `LatestWins`（仅保留最新帧以最小化延迟）。

### `VideoEncodeSource`
编码输入源枚举：
- **`Camera { input_id, format_id }`**：摄像头实时采集。
- **`Texture { texture_id }`**：GPU 纹理输入。
- **`CpuFrames { layout }`**：CPU 端帧数据输入。

### `VideoEncoderConfig`
编码器完整配置：编解码器、输入源、分辨率、帧率、目标码率、关键帧间隔、实时低延迟模式、编码模式参数（用于 AV1 软件编码质量/速度调节）、队列策略与队列容量。

### `EncodedVideoPacketRef<'a>`
编码后的输出包引用，包含编码器、码流格式、PTS/DTS 时间戳、关键帧标记、配置帧标记（SPS/PPS/序列头）、结束标记、配置 ID、以及数据切片。
- `is_config` 表示编解码器参数载荷，对应特定的 `config_id`。
- `is_key` 表示可独立解码的关键帧。
- `config_id` 单调递增，编码器参数变更时自增。

### `EncodedVideoPacketOwned`
与 `EncodedVideoPacketRef` 对应的是拥有所有权的编码包。

---

## 视频解码器配置

### `VideoDecodeOutput`
解码输出目标枚举：
- **`Texture { texture_id }`**：输出到 GPU 纹理。
- **`YuvPlanes { tex_y, tex_u, tex_v }`**：输出到三平面 YUV 纹理。
- **`CpuFrames`**：输出到 CPU 端帧。

### `VideoDecoderConfig`
解码器配置：编解码器、期望码流格式、输出目标、宽度/高度提示、实时低延迟模式。

### `VideoDecoderPacketRef<'a>`
解码器输入包引用，包含 PTS/DTS、关键帧/配置帧标记、配置 ID、数据切片。解码器协议要求：解码器必须先接收配置帧（`is_config=true`）才能解码后续依赖帧；`config_id` 变化表示流重新配置。

### `VideoDecodeError`
解码器错误枚举：`UnsupportedCodec`、`UnsupportedOutput`、`DecoderNotStarted`、`InvalidPacket`。

---

## 编解码能力查询

### `VideoCodecSupport`
描述某个编解码器的完整能力集：软/硬件编解码支持、支持的码流格式列表、摄像头/纹理/CPU帧输入支持、关键帧请求支持、动态分辨率支持、对齐约束、最大分辨率/帧率/码率限制。

`unsupported(codec)` 构造函数创建全 false 的不可用描述。

### `VideoCapabilities`
包含 `Vec<VideoCodecSupport>` 的能力向量，表示平台整体的编解码能力。

---

## 视频输入与缓冲区

### `VideoBufferRefData<'a>` / `VideoBufferRef<'a>` / `VideoBufferData` / `VideoBuffer`
视频帧数据缓冲区的引用与所有权类型，支持 U8 和 U32 两种数据表示。提供 `to_buffer()`（引用→所有权）、`as_slice_u32/as_slice_u8`、`as_vec_u32/as_vec_u8`、`into_vec_u32/into_vec_u8` 等转换方法。

### `VideoFormat`
视频格式描述：`format_id`、宽高、帧率、像素格式。

### `VideoInputDesc`
视频输入设备描述：设备 ID、名称、支持的格式列表。

### `VideoInputsEvent`
设备枚举事件，包含 `Vec<VideoInputDesc>`。提供多种设备选择策略：
- **`find_device(name)`**：按名称全匹配查找设备索引，未找到则返回 0。
- **`find_highest(device_index)`**：选择指定设备的最高质量格式——先找最大分辨率，再在最大分辨率中找最大帧率，再在同等条件下找最高质量像素格式。
- **`find_highest_at_res(device_index, width, height, max_fps)`**：在给定分辨率下选择最高帧率（不超过 `max_fps`）和最高像素质量的格式。
- **`find_format(device_index, width, height, pixel_format)`**：精确匹配分辨率和像素格式后选择最高帧率。
