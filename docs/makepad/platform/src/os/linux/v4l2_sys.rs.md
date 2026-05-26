# v4l2_sys.rs

One-liner (EN): FFI type definitions, ioctl constants, and function declarations for the Linux V4L2 (Video4Linux2) camera API.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/v4l2_sys.rs` (342 行)
- **核心作用**: 提供 Linux V4L2 视频采集子系统的 Rust FFI 绑定，包括 ioctl 命令码、V4L2 数据结构体定义、poll/inotify 系统调用声明。

## 关键常量

### V4L2 能力标志
| 常量 | 值 | 说明 |
|------|-----|------|
| `V4L2_CAP_VIDEO_CAPTURE` | `0x00000001` | 支持视频捕获 |
| `V4L2_CAP_STREAMING` | `0x04000000` | 支持流式 I/O |
| `V4L2_CAP_DEVICE_CAPS` | `0x80000000` | 支持设备能力查询 |

### 缓冲/内存类型
| 常量 | 值 | 说明 |
|------|-----|------|
| `V4L2_BUF_TYPE_VIDEO_CAPTURE` | 1 | 视频捕获缓冲区类型 |
| `V4L2_MEMORY_MMAP` | 1 | 内存映射缓冲区 |
| `V4L2_FIELD_ANY` | 0 | 任意场序 |

### 像素格式 (fourcc)
| 常量 | 值 | 说明 |
|------|-----|------|
| `V4L2_PIX_FMT_YUYV` | `YUYV` | YUYV 4:2:2 打包格式 |
| `V4L2_PIX_FMT_MJPEG` | `MJPEG` | 运动 JPEG 压缩格式 |
| `V4L2_PIX_FMT_NV12` | `NV12` | YUV 4:2:0 平面格式 |
| `V4L2_PIX_FMT_YUV420` | `YU12` | I420 平面格式 |
| `V4L2_PIX_FMT_RGB24` | `RGB3` | 24 位 RGB 打包格式 |
| `V4L2_PIX_FMT_GREY` | `GREY` | 8 位灰度格式 |

### ioctl 命令常量 (通过 `ioc()` 宏计算)
| 常量 | 说明 |
|------|------|
| `VIDIOC_QUERYCAP` | 查询设备能力 |
| `VIDIOC_ENUM_FMT` | 枚举支持的像素格式 |
| `VIDIOC_G_FMT / S_FMT` | 获取/设置当前格式 |
| `VIDIOC_REQBUFS` | 请求缓冲区 |
| `VIDIOC_QUERYBUF` | 查询缓冲区信息 |
| `VIDIOC_QBUF / DQBUF` | 入队/出队缓冲区 |
| `VIDIOC_STREAMON / STREAMOFF` | 启动/停止视频流 |
| `VIDIOC_S_PARM` | 设置流参数（帧率） |
| `VIDIOC_ENUM_FRAMESIZES` | 枚举帧尺寸 |
| `VIDIOC_ENUM_FRAMEINTERVALS` | 枚举帧间隔 |

### 其他系统常量
| 常量 | 说明 |
|------|------|
| `POLLIN` | 可读事件标志 |
| `IN_CREATE / IN_DELETE` | inotify 文件创建/删除事件 |
| `IN_NONBLOCK` | 非阻塞打开标志 (04000 octal) |

## 关键类型/结构体

| 结构体 | 说明 |
|--------|------|
| `v4l2_capability` | 设备能力（driver, card, bus_info, capabilities 等） |
| `v4l2_fmtdesc` | 格式描述（index, type, flags, description, pixelformat） |
| `v4l2_fract` | 分数表示（numerator/denominator） |
| `v4l2_pix_format` | 像素格式（width, height, pixelformat, bytesperline, sizeimage） |
| `v4l2_format` | 格式信息（type + 联合体 `v4l2_format_fmt`） |
| `v4l2_requestbuffers` | 缓冲区请求（count, type, memory） |
| `v4l2_buffer` | 缓冲区信息（index, bytesused, timestamp, memory, m.offset 等） |
| `v4l2_frmsizeenum` | 帧尺寸枚举（discrete/stepwise） |
| `v4l2_frmivalenum` | 帧间隔枚举（discrete/stepwise） |
| `v4l2_streamparm` | 流参数（包含 capture parm: capability, timeperframe） |
| `pollfd` | poll 系统调用的文件描述符结构 |
| `inotify_event` | inotify 事件结构 |

## FFI 函数声明
| 函数 | 说明 |
|------|------|
| `ioctl(fd, request, arg)` | V4L2 设备控制命令 |
| `poll(fds, nfds, timeout)` | I/O 多路复用 |
| `inotify_init1(flags)` | 创建 inotify 实例 |
| `inotify_add_watch(fd, pathname, mask)` | 添加 inotify 监视 |

## 实现细节
- 辅助宏 `ioc()` 构造 V4L2 ioctl 命令码（_IOW/_IOR 风格），使用 `dir << 30 | type << 8 | nr << 0 | size << 16` 布局
- 辅助宏 `fourcc()` 将 4 字符代码编码为 u32
- 联合体 `v4l2_format_fmt` 和 `v4l2_buffer_m` 使用 `#[repr(C)]` 和显式对齐指针以兼容 32/64 位架构
- `pollfd` 结构体独立于 libc 的 `pollfd`，避免对外部 crate 的依赖
