# `platform/src/script/res.rs` — 资源加载系统

## 文件职责

该文件实现了 Makepad 平台层与 Splash VM 之间的资源加载集成。它定义了资源数据结构（`CxScriptResource`、`CxScriptResources`、`CxScriptHttpResource`）、多平台资源加载策略（文件系统/打包资源/HTTP）、以及脚本 API 方法（`res.file()`、`res.crate()`、`res.http_resource()`、`res.binary_resource()`、`res.load_all_resources()`）。

---

## 核心数据结构

### `CxScriptResourceData`

资源加载状态的枚举：
- `NotLoaded`：尚未开始加载
- `Loading`：加载中（用于 HTTP 异步请求）
- `Loaded(Rc<Vec<u8>>)`：加载完成，持有字节数据的引用计数指针
- `Error(String)`：加载失败，携带错误信息

使用 `Rc<Vec<u8>>` 允许多个持有者共享同一份字节数据，避免拷贝。

### `CxScriptResource`

单个脚本资源：
- `abs_path: String` — 资源的绝对路径（也用作唯一标识符）
- `dependency_path: Option<String>` — 相对于包根目录的依赖路径
- `web_url: Option<String>` — Web 平台上的 HTTP URL
- `data: CxScriptResourceData` — 当前加载状态和数据
- `handle: ScriptHandle` — 脚本 VM 中的句柄引用

提供 `is_error()` 方法快速判断加载是否失败。

### `CxScriptHttpResource`

跟踪一个正在进行的 HTTP 请求：
- `request_id: LiveId` — HTTP 请求的唯一 ID，用于关联响应
- `handle: ScriptHandle` — 对应的资源句柄

### `CxScriptResources`

资源集合的管理容器：
- `resources: Rc<RefCell<Vec<CxScriptResource>>>` — 所有资源的共享列表
- `handles_by_abs_path: Rc<RefCell<HashMap<String, ScriptHandle>>>` — 绝对路径到句柄的映射
- `http_resources: Vec<CxScriptHttpResource>` — 正在进行的 HTTP 请求列表

提供的方法：
- `get_handle_by_abs_path()`：通过绝对路径查找已有的句柄，避免重复创建
- `insert_resource()`：插入新资源并更新路径→句柄映射
- `get_data()`：根据句柄获取已加载的字节数据
- `handle_http_response()`：根据 `request_id` 查找并更新 HTTP 响应数据
- `handle_http_error()`：根据 `request_id` 标记 HTTP 错误
- `is_http_resource()`：判断某个 `request_id` 是否属于 HTTP 资源请求

### `CxScriptResourceGc`

实现 `ScriptHandleGc` trait 的垃圾回收辅助结构。当脚本句柄被 GC 回收时：
1. 从 `resources` 列表中移除对应的资源条目
2. 从 `handles_by_abs_path` 映射中清除对应的路径→句柄映射

---

## 平台资源加载策略

详细的跨平台加载逻辑（代码注释第 121–144 行）：

| 平台 | 加载顺序 |
|------|---------|
| **Desktop 非打包** | ① 依赖表 → ② 绝对路径文件系统 → ③ 错误 |
| **Desktop 打包** | ① 依赖表 → ② `package_root/dep_path` 文件系统 → ③ 错误 |
| **iOS/tvOS** | ① 依赖表 → ② `NSBundle.resourcePath/package_root/dep_path` → ③ 错误 |
| **Android** | ① 依赖表（`get_dependency` 调用 JNI 资源管理器）→ ② 错误 |
| **Wasm** | ① 依赖表（预加载依赖）→ ② HTTP 异步获取 → ③ 错误 |

### `load_packaged_resource(cx, dep_path)`（Apple）

- iOS/tvOS 和 macOS 捆绑包专有
- 拼接 `package_root/dep_path`，调用 `cx.apple_bundle_load_file()` 通过 NSBundle 加载

### `load_packaged_resource(cx, dep_path)`（Desktop）

- 非 Wasm、非 Apple 打包桌面专有
- 检查 `package_root` 是否存在，若存在则直接通过文件系统读取

### `load_file_direct(abs_path)`

- 所有非 Wasm 平台通用
- 直接通过 `File::open` 和 `read_to_end` 读取文件
- 成功返回 `Ok(Rc<Vec<u8>>)`，失败返回 `Err(String)`

### `should_skip_eager_resource_load(abs_path)`

判断是否应该跳过主动加载。当前逻辑是：如果路径匹配一个**重型字体回退文件**则跳过。重型字体包括：
- `LXGWWenKaiRegular.ttf`（中文楷体约 15MB）
- `LXGWWenKaiBold.ttf`（中文粗楷体）
- `NotoColorEmoji.ttf`（彩色 emoji 字体）

这些字体体积大，只在需要时才按需加载，避免启动时不必要的 I/O。

### `is_widgets_resources_path(path)`

检查路径是否形如 `.../widgets/resources/...`，路径组件以 `/` 或 `\` 分割。

### `resource_basename(path)`

提取路径最后一个 `/` 或 `\` 后的文件名。

---

## Web 平台路径解析

### `remapped_small_font_dependency_path(path)`

在测试和 Wasm 平台下，将大型 fallback 字体路径重映射到更小的 IBM Plex Sans 字体，减小 Web 下载体积：
- 五个重型字体路径映射到两个较小的字体文件

### `web_resource_request_path(cx, dep_path)`

在 Wasm 上构建 HTTP 请求路径：
- 拼接 `base_path/dep_path`
- 如果 `small_font_aliases` 标志启用，先进行字体路径重映射

### `web_resource_base_path(pathname)`

从浏览器 URL pathname 推断资源基础路径：
- 处理形如 `/makepad-example-splash/index.html` → `makepad-example-splash`
- 去掉尾部的文件名（最后一个包含 `.` 的段）

---

## `Cx::load_script_resource_impl()`

资源加载的核心实现（`res.rs:284`）：

1. **Wasm 护身**：如果 `os_type` 为 `Unknown`（尚未收到初始化消息），跳过
2. **获取资源条目**：通过句柄在 `resources` 列表中定位
3. **状态检查**：如果不是 `NotLoaded` 状态，直接返回（幂等性保证）
4. **Wasm 依赖路径解析**：如果 `dependency_path` 为空，通过 crate manifests 尝试解析
5. **Wasm URL 构建**：设置 `web_url` 用于 HTTP 获取
6. **依赖表尝试**：如果存在 `dependency_path`，先尝试从已注册的依赖表中获取数据
7. **Wasm HTTP 回退**：Wasm 上如果依赖表没有，设置状态为 `Loading` 并发起 HTTP 请求
8. **非 Wasm 回退**：
   - 打包模式：调用 `load_packaged_resource()`
   - 非打包模式：调用 `load_file_direct()` 通过绝对路径读取
9. **错误处理**：所有路径都失败后，标记为 `Error` 并记录描述信息

### `Cx::load_script_resource(handle)`

公共包装方法，在 Wasm 上预先克隆 `crate_manifests`，然后调用 `load_script_resource_impl`。

### `Cx::load_all_script_resources()`

遍历所有资源，过滤掉 `should_skip_eager_resource_load` 返回 `true` 的项（重型字体），依次调用 `load_script_resource_impl`。

---

## Crate 资源路径解析

### `parse_crate_path(path)`

解析 `crate_name:path/to/file` 格式的字符串，返回 `(crate_part, file_path)` 元组。

### `strip_crate_resource_leading_slashes(file_path)`

去除 `self://path` 或 `self:path` 开头的多余斜杠。

### `normalize_dependency_file_path(path)`

将路径标准化处理，解析 `.`、`..` 段，统一为 `/` 分隔符。

### `normalize_path(path)`（Wasm 专有）

`PathBuf` 标准化工具，处理 `..`、`.` 和 `Prefix` 组件。

### `normalize_manifest_relative_path(path)`（Wasm 专有）

将 Cargo manifest 的相对路径标准化为依赖路径格式。

### `resolve_dependency_path_from_manifests()`（Wasm 专有）

根据绝对路径和已知的 crate manifests 表，反向计算出依赖路径：
1. 标准化绝对路径
2. 收集所有候选（默认 crate + manifests 表中的所有 crate）
3. 对每个候选，尝试将绝对路径减去 manifest 所在目录，得到相对路径
4. 选择匹配路径最长的候选（最精确匹配）
5. 返回 `crate_name/relative_path` 格式的依赖路径

### `resolve_crate_resource_paths()`（两套实现）

**Wasm 版本**（`res.rs:642`）：
- 根据 `crate_part` 是否为 `"self"`，从当前脚本模块的 `cargo_manifest_path` 或从 `crate_manifests` 表中获取绝对路径
- 计算 `dependency_path` 和 `web_url`
- 如果未能映射出依赖路径，记录警告日志

**非 Wasm 版本**（`res.rs:711`）：
- 同样处理 `"self"` 与其他 crate 分支
- 直接拼接 `crate_name/relative_path` 作为依赖路径
- `web_url` 始终为 `None`

---

## 脚本 API 方法注册 (`pub fn script_mod`)

### `res` 模块和句柄类型创建

```rust
let res = vm.new_module(id!(res));
let res_type = vm.new_handle_type(id_lut!(res));
```

创建 `res` 脚本模块和 `res` 句柄类型。

### 资源句柄属性 getter

通过 `vm.set_handle_getter()` 注册资源句柄的七个属性访问器：
- **`path`**：返回资源的绝对路径字符串
- **`is_loaded`**：返回布尔值，表示资源是否已成功加载
- **`is_error`**：返回布尔值，表示资源是否加载失败
- **`error`**：如果加载失败，返回错误字符串；否则返回 `NIL`
- **`data`**：如果加载成功，返回 `U8` 数组的字节数据；否则返回 `NIL`
- 对于非法属性名，通过 `script_err_not_found!` 触发脚本错误

### `res.load_all_resources(value)` → 原始值

- 调用 `cx.load_all_script_resources()` 触发所有待加载资源的加载
- 返回传入的 `value` 参数（管道模式，可以链式传递）

### `res.file_resource(path)` → 句柄

- 接受一个字符串参数作为绝对文件路径
- 如果该路径已有对应的资源句柄，直接返回（去重）
- 否则创建新的 `CxScriptResourceGc`，通过 `vm.bx.heap.new_handle()` 分配句柄
- 插入资源并设置状态为 `NotLoaded`
- 返回句柄

### `res.crate_resource(path)` → 句柄

- 接受 `"crate:path"` 格式的字符串
- 解析 crate 部分和文件路径部分
- 调用 `resolve_crate_resource_paths()` 获取绝对路径、依赖路径和 Web URL
- 如果路径重复，返回已存在的句柄
- 否则创建新资源并插入，状态为 `NotLoaded`

### `res.http_resource(url)` → 句柄

- 接受 HTTP/HTTPS URL 字符串
- 如果 URL 已有对应句柄，直接返回
- 创建新资源，状态直接设为 `Loading`
- 立即通过 `cx.http_request()` 发起 HTTP GET 请求
- 将 `(request_id, handle)` 记录到 `http_resources` 跟踪列表
- 返回句柄（脚本代码可以轮询 `is_loaded` 或 `is_error`）

### `res.binary_resource(data)` → 句柄

- 接受一个 `U8` 字节数组
- 创建一个 `binary://` 伪路径的资源
- 数据直接设置为 `Loaded` 状态，无需 I/O
- 适用于脚本构造的纹理或文件数据

---

## 辅助函数

### `script_value_to_u8_bytes(vm, value)`

从脚本值中提取 `Vec<u8>` 字节数据。检查值是否为数组且存储类型为 `U8`。

---

## 测试

`mod tests` 包含三个单元测试：
1. `skips_only_heavy_widgets_fallback_fonts`：验证只有 widgets 目录下的三大重型字体被跳过
2. `remaps_only_known_small_font_fallback_dependency_paths`：验证正确的字体路径映射
3. `derives_web_resource_base_path_from_browser_pathname`：验证 Web 基础路径推导逻辑
