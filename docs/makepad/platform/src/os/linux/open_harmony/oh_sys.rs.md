# oh_sys.rs

One-liner (EN): FFI bindings to OpenHarmony native APIs (VSync, libuv work queue, rawfile resource manager).

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/open_harmony/oh_sys.rs` (95 行)
- **核心作用**: 提供 OpenHarmony 系统级 FFI 绑定，包括垂直同步（VSync）、libuv 工作队列（用于在主线程调用 JS 函数）和原生资源文件（Rawfile）管理。

## FFI 绑定

### OH_NativeVSync — 垂直同步

```c
// C 原型（通过 extern "C" 绑定）
OH_NativeVSync* OH_NativeVSync_Create(const char *name, unsigned int length);
void OH_NativeVSync_Destroy(OH_NativeVSync *nativeVsync);
int OH_NativeVSync_RequestFrame(OH_NativeVSync *nativeVsync,
                                void (*callback)(long long, void*),
                                void *data);
```

- `OH_NativeVSync` — 不透明结构体（零大小类型，仅用作指针标记）
- 链接库: `native_vsync`

### libuv — 工作队列

| 类型/常量 | 源定义 |
|-----------|--------|
| `uv_loop_t` | 别名 `napi_ohos::sys::uv_loop_s` — 事件循环句柄 |
| `uv_req_type` | `u32` — 请求类型枚举 |
| `uv_work_t` / `uv_work_s` | 工作请求结构体，包含 `data`, `work_cb`, `after_work_cb` 等字段 |
| `uv__work` | libuv 内部工作结构（`work`/`done` 回调 + `wq` 双链表） |

```c
int uv_queue_work(uv_loop_t *loop, uv_work_t *req,
                  uv_work_cb work_cb, uv_after_work_cb after_work_cb);
```

- `uv_work_cb` — 在**线程池**中执行的回调（类型: `Option<unsafe extern "C" fn(*mut uv_work_t)>`）
- `uv_after_work_cb` — 在主循环中执行的回调，用于调用 JS 函数
- 链接库: `uv`

### Rawfile — 原生资源文件

| 类型 | 说明 |
|------|------|
| `RawFile` | 不透明结构体，代表已打开的资源文件句柄 |
| `NativeResourceManager` | 不透明结构体，代表原生资源管理器 |

```c
NativeResourceManager* OH_ResourceManager_InitNativeResourceManager(napi_env, napi_value);
void OH_ResourceManager_ReleaseNativeResourceManager(NativeResourceManager*);
RawFile* OH_ResourceManager_OpenRawFile(const NativeResourceManager*, const char*);
long OH_ResourceManager_GetRawFileSize(RawFile*);
void OH_ResourceManager_CloseRawFile(RawFile*);
int OH_ResourceManager_ReadRawFile(const RawFile*, void*, unsigned long);
```

- 链接库: `rawfile.z`

### 链接库汇总

| 库名 | 用途 |
|------|------|
| `ace_napi.z` | ACE napi 运行时 |
| `ace_ndk.z` | ACE NDK 支持 |
| `hilog_ndk.z` | HiLog 日志 |
| `native_window` | 原生窗口操作 |
| `native_vsync` | 垂直同步 |
| `uv` | libuv 事件循环 |
| `rawfile.z` | 原生资源文件 |

## 实现细节

- 所有 C 回调类型使用 `Option<unsafe extern "C" fn(...)>` 以支持空指针安全
- 不透明结构体使用 `#[repr(C)]` + `_unused: [u8; 0]` 零大小标记类型（仅作指针类型安全用）
