# Linux DRM FFI 绑定

## 概述

`drm_sys.rs` 是 Linux Direct Rendering Manager (DRM) API 的 FFI 绑定。用于 KMS (Kernel Mode Setting) 和翻页显示。

通过 `#[link(name = "drm")]` 链接 `libdrm`。

## 不透明类型

| 类型 | 说明 |
|------|------|
| `drmDevice` / `_drmDevice` | DRM 设备描述 |
| `drmModeRes` / `_drmModeRes` | KMS 资源（CRTC、编码器、连接器） |
| `drmModeConnector` / `_drmModeConnector` | 显示连接器 |
| `drmModeEncoder` / `_drmModeEncoder` | 编码器 |
| `drmModeModeInfo` / `_drmModeModeInfo` | 显示模式 |
| `drmEventContext` / `_drmEventContext` | 事件处理上下文 |

## 总线/设备信息结构体

| 类型 | 说明 |
|------|------|
| `_drmPciBusInfo` | PCI 总线信息（domain/bus/dev/func） |
| `_drmPciDeviceInfo` | PCI 设备信息（vendor/device/subvendor/subdevice/revision） |
| `_drmUsbBusInfo` | USB 总线信息（bus/dev） |
| `_drmUsbDeviceInfo` | USB 设备信息（vendor/product） |
| `_drmPlatformBusInfo` / `_drmHost1xBusInfo` | 平台/主机总线信息（fullname） |
| `_drmPlatformDeviceInfo` / `_drmHost1xDeviceInfo` | 平台/主机设备信息（compatible） |
| `_drmDevice` | 完整设备描述（nodes、available_nodes、bustype、businfo、deviceinfo 联合体） |

## 显示结构体

| 类型 | 说明 |
|------|------|
| `_drmModeRes` | 资源（FBs、CRTCs、连接器、编码器数量及 ID 数组，min/max 尺寸） |
| `_drmModeConnector` | 连接器（connector_id、encoder_id、连接状态、模式列表、编码器列表） |
| `_drmModeEncoder` | 编码器（encoder_id、type、crtc_id、possible_crtcs/clones） |
| `_drmModeModeInfo` | 显示模式（clock、h/vdisplay、h/vsync_start/end、h/ vtotal、vrefresh、flags、name） |
| `_drmEventContext` | 事件处理（vblank、page_flip 处理器） |

## 外部函数

### 设备枚举
- `drmGetDevices2` — 获取所有 DRM 设备

### 资源查询
- `drmModeGetResources` — 获取 KMS 资源
- `drmModeGetConnector` — 获取连接器
- `drmModeGetEncoder` — 获取编码器
- `drmModeFreeConnector` / `drmModeFreeResources` / `drmModeFreeEncoder` — 释放资源

### 帧缓冲区
- `drmModeAddFB2` — 添加帧缓冲区（指定格式、句柄、stride、偏移量）

### 模式设置
- `drmModeSetCrtc` — 设置 CRTC（绑定帧缓冲区到连接器）

### 翻页
- `drmModePageFlip` — 请求翻页（支持 `PAGE_FLIP_EVENT` 和 `PAGE_FLIP_ASYNC`）

### 事件处理
- `drmHandleEvent` — 处理 DRM 事件（读取翻页完成等）

## 常量

- `MAX_DRM_DEVICES = 64` — 最大设备枚举数
- `DRM_NODE_PRIMARY = 0` — 主节点
- `DRM_MODE_CONNECTED = 1` — 连接器状态：已连接
- `DRM_MODE_PAGE_FLIP_EVENT = 1` — 翻页事件标志
- `DRM_MODE_PAGE_FLIP_ASYNC = 2` — 异步翻页标志

## 类型别名

所有 `_Xxx` 原始结构体都有对应的 `XxxPtr` 类型别名（如 `drmModeConnectorPtr`），用于外部函数签名。
