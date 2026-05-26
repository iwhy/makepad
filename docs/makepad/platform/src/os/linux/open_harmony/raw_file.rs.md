# raw_file.rs

One-liner (EN): Wrapper around the OpenHarmony native resource manager (Rawfile) for reading bundled assets.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/open_harmony/raw_file.rs` (68 行)
- **核心作用**: 封装 OpenHarmony 的原生资源文件（Rawfile）API，提供从应用 bundle 中读取资源文件的能力。通过 `OH_ResourceManager_*` FFI 函数实现文件的打开、读取、关闭操作。

## 类型/结构体

### `RawFileMgr`

| 字段 | 类型 | 说明 |
|------|------|------|
| `native_resource_manager` | `*mut NativeResourceManager` | OpenHarmony 原生资源管理器指针 |

实现了 `Clone + Debug` trait。

## 关键方法

| 方法 | 签名 | 说明 |
|------|------|------|
| `new(raw_env, res_mgr)` | `(napi_env, napi_value) -> Self` | 从 napi 值初始化原生资源管理器 |
| `read_to_end(path, buf)` | `(&mut self, impl AsRef<str>, &mut Vec<u8>) -> Result<usize>` | 读取指定路径的资源文件到缓冲区，返回读取字节数 |
| `drop` | 析构 | 释放原生资源管理器 |

## 实现细节

### `read_to_end` 流程

1. **空检查**: 若 `native_resource_manager` 为 null，返回 `ErrorKind::NotConnected`
2. **打开文件**: 调用 `OH_ResourceManager_OpenRawFile` 获取 `RawFile` 指针
3. **获取大小**: 调用 `OH_ResourceManager_GetRawFileSize`
4. **分配缓冲区**: 根据文件大小调整 `Vec<u8>` 容量
5. **读取内容**: 调用 `OH_ResourceManager_ReadRawFile` 将文件数据读入缓冲区
6. **处理截断**: 若实际读取字节数小于文件大小，缩减缓冲区
7. **关闭文件**: 调用 `OH_ResourceManager_CloseRawFile`
8. **返回结果**: 返回读取的字节数

### 资源管理

- `NativeResourceManager` 通过 `OH_ResourceManager_InitNativeResourceManager` 从 napi 值创建
- 在 `Drop` 中通过 `OH_ResourceManager_ReleaseNativeResourceManager` 释放
- 每次 `read_to_end` 调用都会打开和关闭文件，不缓存文件描述符
- 使用 `std::io::Result` 类型，兼容 Rust 标准 I/O 错误处理
