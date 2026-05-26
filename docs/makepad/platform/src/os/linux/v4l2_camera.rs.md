# v4l2_camera.rs

One-liner (EN): V4L2 camera device management — device probing, format negotiation, mmap streaming capture, and frame delivery via callbacks.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/v4l2_camera.rs` (945 行)
- **核心作用**: 实现 Linux V4L2 摄像头的完整捕获管线：设备探测（枚举 /dev/videoN 节点）、格式协商（逐级降级匹配）、MMAP 缓冲区分配与队列管理、poll+DQBUF 捕获循环、帧数据分发到视频输入回调和编码器。

## 类型/结构体

### `V4l2CameraDevice` — 设备描述

| 字段 | 类型 | 说明 |
|------|------|------|
| `path` | `String` | 设备路径（如 `/dev/video0`） |
| `desc` | `VideoInputDesc` | 视频输入描述（含格式列表） |

### `MmapBuffer` — MMAP 缓冲区

| 字段 | 类型 | 说明 |
|------|------|------|
| `ptr` | `*mut u8` | mmap 映射的缓冲区指针 |
| `length` | `usize` | 缓冲区长度 |

实现了 `unsafe impl Send` 以在线程间传递。

### `SendPtr` — 可发送的原始指针包装

`struct SendPtr(*mut u8)` + `unsafe impl Send` — 用于向捕获线程传递缓冲区指针列表。

### `V4l2CaptureSession` — 捕获会话

| 字段 | 类型 | 说明 |
|------|------|------|
| `fd` | `c_int` | 设备文件描述符 |
| `running` | `Arc<AtomicBool>` | 运行时标志（线程控制） |
| `thread` | `Option<JoinHandle<()>>` | 捕获线程句柄 |
| `buffers` | `Vec<MmapBuffer>` | mmap 缓冲区列表 |

### `V4l2CameraAccess` — 摄像头访问入口

| 字段 | 类型 | 说明 |
|------|------|------|
| `video_input_cb` | `[Arc<Mutex<Option<VideoInputFn>>>; MAX_VIDEO_DEVICE_INDEX]` | 视频输入回调数组 |
| `camera_frame_input_cb` | `[Arc<Mutex<Option<CameraFrameInputFn>>>; MAX_VIDEO_DEVICE_INDEX]` | 相机帧回调数组 |
| `video_output_cb` | `[Arc<Mutex<Option<VideoOutputFn>>>; MAX_VIDEO_DEVICE_INDEX]` | 视频输出回调数组 |
| `video_encoder_config` | `[Arc<Mutex<Option<VideoEncoderConfig>>>; MAX_VIDEO_DEVICE_INDEX]` | 编码器配置数组 |
| `video_encoder` | `[Arc<Mutex<Option<VideoEncoder>>>; MAX_VIDEO_DEVICE_INDEX]` | 编码器实例数组 |
| `devices` | `Vec<V4l2CameraDevice>` | 探测到的设备列表 |
| `sessions` | `Vec<V4l2CaptureSession>` | 活跃的捕获会话 |

## 关键方法

### V4l2CaptureSession

| 方法 | 说明 |
|------|------|
| `start(input_fn, frame_input_fn, video_encoder, device_path, format) -> Option<Self>` | 启动捕获会话：打开设备→协商格式→设置格式→请求 MMAP 缓冲区→查询映射→入队所有缓冲区→STREAMON→启动捕获线程 |
| `negotiate_format(fd, requested) -> Option<VideoFormat>` | 格式协商：枚举设备支持的所有格式/尺寸/帧率，按优先级匹配（精确匹配→同格式最接近分辨率→YUYV/MJPEG/NV12/YUV420 优选→任意） |
| `pick_closest_resolution(supported, pixfmt, target_w, target_h)` | 从匹配指定像素格式的条目中选择最接近目标分辨率的组合 |
| `capture_loop(fd, format, input_fn, frame_input_fn, video_encoder, buffers, running)` | 捕获线程主循环：poll 等待数据→DQBUF→解析帧格式（YUV420/NV12/YUY2/MJPEG）→分发给 frame_input_cb 和 video_encoder→分发给 input_cb→QBUF 重新入队 |
| `stop()` | 停止捕获：设置 running=false→join 线程→STREAMOFF→munmap→close |

### V4l2CameraAccess

| 方法 | 说明 |
|------|------|
| `new(change_signal) -> Arc<Mutex<Self>>` | 创建访问入口，启动设备热插拔监视线程（优先 inotify，回退轮询） |
| `use_video_input(inputs)` | 配置使用指定输入：停止所有旧会话→清空编码器→为每个输入启动新的 V4l2CaptureSession（含视频编码器初始化） |
| `get_updated_descs() -> Vec<VideoInputDesc>` | 重新探测 /dev/video0~63 设备，收集支持的格式/分辨率/帧率，返回描述列表 |
| `probe_device(path) -> Option<V4l2CameraDevice>` | 探测单个 V4L2 设备 |
| `probe_device_fd(fd, path) -> Option<V4l2CameraDevice>` | 查询设备能力→枚举像素格式→枚举帧尺寸→枚举帧间隔→构建 VideoInputDesc |
| `watch_devices(change_signal)` | **设备热插拔监视**：inotify 监视 /dev 目录的 CREATE/DELETE 事件，检测 video 设备变化后触发 change_signal |
| `poll_devices(change_signal)` | 回退方案：每 2 秒轮询 /dev/videoN 的存在数量，变化时触发信号 |

### 辅助函数

| 函数 | 说明 |
|------|------|
| `fourcc_to_video_pixel_format(fourcc)` | V4L2 fourcc → VideoPixelFormat 枚举 |
| `video_pixel_format_to_fourcc(format)` | VideoPixelFormat → V4L2 fourcc |
| `cstr_from_bytes(bytes)` | C 风格字节数组（V4L2 设备名）→ Rust String（UTF-8 lossy） |

## 实现细节

### 捕获线程架构

```
主线程                         捕获线程
  │                              │
  │ use_video_input()            │
  │   └ start()                  │
  │     ├ 打开 /dev/videoN       │
  │     ├ 格式协商               │
  │     ├ VIDIOC_S_FMT           │
  │     ├ VIDIOC_REQBUFS (4 buf) │
  │     ├ mmap 所有缓冲区         │
  │     ├ VIDIOC_QBUF x4         │
  │     ├ VIDIOC_STREAMON        │
  │     └  spawn 捕获线程 ────────► │
  │                              │ while running:
  │                              │   poll(fd, timeout=200ms)
  │                              │   VIDIOC_DQBUF
  │                              │   解析帧格式
  │                              │   → frame_input_cb (I420/裁剪)
  │                              │   → video_encoder
  │                              │   → input_cb (原始数据)
  │                              │   VIDIOC_QBUF (重新入队)
```

### 格式协商策略（`negotiate_format`）

1. 枚举设备所有支持的 (pixelformat, width, height) 组合
2. **精确匹配**：请求的格式 + 分辨率
3. **同格式不同分辨率**：欧几里得距离最接近
4. **不同格式**：按 YUYV → MJPEG → NV12 → YUV420 优先级查找
5. **降级兜底**：取设备第一个可用格式
6. 若枚举无结果，使用 `VIDIOC_G_FMT` 获取当前格式

### 帧格式解析（`capture_loop`）

| 像素格式 | 布局 | Plane 数量 | 说明 |
|----------|------|-----------|------|
| `YUV420` (I420) | Y + U + V 三个平面 | 3 | Y 平面 width×height, UV 各 (w/2)×(h/2) |
| `NV12` | Y + UV 交错平面 | 2 | UV 平面为交错 CbCr，像素跨度 2 |
| `YUY2` (YUYV) | 打包式 YUYV | 1 | 每像素 2 字节，Y 和 UV 交错 |
| `MJPEG` | JPEG 压缩流 | 1 | 原始压缩数据，不解析 |

### 设备热插拔监视

- **首选**: inotify 监视 `/dev` 目录的 `IN_CREATE|IN_DELETE`，过滤 `video` 开头的文件，检测到变化后延迟 500ms 触发信号
- **回退**: 每 2 秒轮询 `/dev/videoN` (0~63) 的存在数量，变化时触发信号
- 触发信号类型: `SignalToUI::set()` — 通知主线程重新枚举设备

### 视频编码器集成

当 `video_output_cb` 存在且有配置时，`use_video_input` 会创建 `VideoEncoder` 实例，将捕获帧同时送入编码器管线。配置使用 `VideoEncodeSource::Camera` 源类型。
