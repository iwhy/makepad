# arkts_obj_ref.rs

One-liner (EN): Wrapper around a napi_ref to an ArkTS object, enabling property reads and asynchronous JS function calls from Rust via libuv work queue.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/open_harmony/arkts_obj_ref.rs` (196 行)
- **核心作用**: 封装对 ArkTS（JavaScript）对象的 `napi_ref` 引用，提供属性读取（字符串、数值、通用属性）和跨线程异步 JS 函数调用能力。JS 函数调用通过 libuv `uv_queue_work` 委托到主线程执行。

## 类型/结构体

### `ArkTsObjErr` — 错误枚举

| 变体 | 说明 |
|------|------|
| `NullError` | CString 创建时遇到空字节 |
| `InvalidProperty` | 属性不存在或为 `undefined` |
| `InvalidStringValue` | 属性值非字符串 |
| `InvalidNumberValue` | 属性值非数字 |
| `InvalidFunction` | 属性值非函数 |
| `CallJsFailed` | `napi_call_function` 调用失败 |

实现了 `From<NulError>` 以便使用 `?` 操作符。

### `ArkTsObjRef` — ArkTS 对象引用

| 字段 | 类型 | 说明 |
|------|------|------|
| `raw_env` | `napi_env` | napi 环境（与 JS 线程关联） |
| `obj_ref` | `napi_ref` | JS 对象的全局引用 |
| `uv_loop` | `*mut uv_loop_t` | libuv 事件循环（用于调度 work） |
| `val_tx / val_rx` | `mpsc::Sender/Receiver` | 用于从 libuv after_work_cb 向调用线程返回结果 |
| `fn_name / argc / argv` | 调用参数 | 暂存下一次 JS 调用的函数名和参数 |
| `worker` | `*mut uv_work_t` | 预分配的 libuv 工作请求结构体 |

## 关键方法

### 构造/生命周期

| 方法 | 签名 | 说明 |
|------|------|------|
| `new(env, obj)` | `(napi_env, napi_ref) -> Self` | 创建引用封装，预分配 uv_work_t（在堆上）并建立 mpsc 通道 |
| `drop` | 析构 | 回收预分配的 uv_work_t 堆内存 |

### 属性读取（可直接调用，无需 JS 线程同步）

| 方法 | 说明 |
|------|------|
| `get_property(name)` | 获取 JS 对象的属性值（`napi_value`）|
| `get_string(name)` | 获取字符串属性 |
| `get_number(name)` | 获取数值属性（`f64`）|
| `get_ref_value()` | 内部方法，将 `napi_ref` 解析为 `napi_value` |

### JS 函数调用（异步，通过 libuv 工作队列）

| 方法 | 签名 | 说明 |
|------|------|------|
| `call_js_function(name, argc, argv)` | `(&mut self, &str, usize, *const napi_value) -> Result<napi_value, ArkTsObjErr>` | 在 JS 主线程上调用对象的方法，通过 mpsc 通道同步等待结果 |

### 辅助

| 方法 | 说明 |
|------|------|
| `raw()` | 返回 `napi_env` |
| `as_ptr()` | 返回指向自身的原始指针（用于 libuv 回调的 `data` 字段） |

## 实现细节

### JS 函数调用流程 (`call_js_function`)

1. 将函数名、参数计数、参数指针写入 `self` 字段
2. 将 `self.as_ptr()` 写入 `worker.data`
3. 调用 `uv_queue_work`：
   - `work_cb` (`js_work_cb`): 不执行任何操作（仅在线程池运行，标记 work 完成）
   - `after_work_cb` (`js_after_work_cb`): 在 JS 主线程上执行：
     - 从反射引用获取 JS 对象
     - 按名称获取函数属性并验证类型
     - 调用 `napi_call_function`
     - 结果通过 `val_tx` 发送回调用线程
4. 调用线程在 `val_rx.recv()` 上阻塞等待结果

### 关键设计

- **预分配的 `uv_work_t`**: 通常 `uv_queue_work` 要求 `uv_work_t` 在调用期间保持有效。预分配避免了每次调用都分配/释放，但也意味着 `ArkTsObjRef` 不是线程安全的（同一时刻只能进行一个 JS 调用）。
- **`get_property` 系列直接调用**: 属性读取不需要切换到 JS 线程，因为 napi 对 `get_reference_value` + `get_named_property` 在任意线程调用是安全的（只要同一时间只有一个线程使用该 env）。
- **错误处理**: 每一步（获取引用、获取属性、验证类型、调用函数）都有详细的错误日志和对应的 `ArkTsObjErr` 错误码。
