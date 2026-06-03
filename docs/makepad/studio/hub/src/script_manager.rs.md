# `script_manager.rs` — 脚本运行时管理器

## 文件位置
- 路径: `studio/hub/src/script_manager.rs`
- 行数: 856 行
- 作用: 管理 `makepad.splash` 脚本的生命周期，提供 RunItem 注册/调用、hub 脚本 API、子进程启动

## 常量

```rust
pub const MAKEPAD_SPLASH_RUNNABLE: &str = "makepad.splash";
```
Splash 脚本文件名，硬编码在挂载根目录。

## 核心数据结构

### `ScriptId`
```rust
pub struct ScriptId(pub u64);
```

### `ScriptCommand`
```rust
enum ScriptCommand {
    RunItem { name: String, child_build_id: QueryId },
}
```

### `ScriptControl`
```rust
struct ScriptControl {
    stop: Arc<AtomicBool>,
    command_tx: Sender<ScriptCommand>,
}
```

### `RunningScript`
```rust
struct RunningScript {
    mount: String,
    control: Arc<ScriptControl>,
}
```

### `RegisteredRunItem`
```rust
struct RegisteredRunItem {
    info: RunItem,
    item: ScriptObjectRef,
}
```

### `ScriptHost`
```rust
struct ScriptHost {
    script_id: ScriptId,
    mount: String,
    cwd: PathBuf,
    studio_local_addr: Option<String>,
    studio_ext_addr: Option<String>,
    event_tx: Sender<HubEvent>,
    stop: Arc<AtomicBool>,
    command_rx: Receiver<ScriptCommand>,
    run_items: HashMap<String, RegisteredRunItem>,
    current_run_item_name: Option<String>,
    current_child_build_id: Option<QueryId>,
}
```
脚本运行时的宿主环境，通过 `downcast_ref` 从脚本 VM 中访问。

## `ScriptManager`

```rust
pub struct ScriptManager {
    next_script_id: u64,
    scripts: HashMap<ScriptId, RunningScript>,
    script_by_mount: HashMap<String, ScriptId>,
}
```

| 方法 | 说明 |
|------|------|
| `start_script(mount, cwd, studio_local_addr, studio_ext_addr, event_tx)` | 为挂载点启动 splash 脚本 |
| `invoke_script_run_item(mount, name, child_build_id)` | 调用已注册的 RunItem |
| `stop_script(script_id)` | 设置 `stop = true` |
| `stop_script_for_mount(mount)` | 按挂载名停止脚本 |
| `mark_exited(script_id, exit_code)` | 移出已退出的脚本，返回 mount |
| `is_running_for_mount(mount)` | 检查是否在运行 |

`alloc_script_id`: 自增 ID，溢出后从 1 重新开始（跳过 0）。

## `run_script_build` — Splash 执行主循环

1. 读取 `makepad.splash` 源文件
2. `normalize_script_source`：确保末尾有 `;`
3. 创建 `ScriptHost`、`NetworkRuntime`、`ScriptStd`、`ScriptVmBase`
4. 编译和执行脚本：
   - 注册 `script_std_mod`
   - 安装 `std.log/print/println` → 转发到 `HubEvent::ScriptOutput`
   - 安装 `hub.*` 模块
5. 进入主循环：
   - 检查 `stop` 标志 → 退出
   - `run_pending_script_commands`（处理 RunItem 调用）
   - `pump`（处理 async 任务）
   - `pump_network_runtime`（处理网络事件）
   - `has_pending_script_work` 检查 → 无工作则退出
   - `sleep(16ms)` ~60fps

## `has_pending_script_work` 检查
检查以下方面是否有待处理工作：
- child_processes（子进程）
- web_sockets（WebSocket 连接）
- http_requests / http_servers
- socket_streams
- tasks.pending_resumes
- 注册的 run items
- 未完成的任务（task）

## 脚本 API — `hub` 模块

### 属性
- `studio_ip` — `String`，hub 的本地地址

### 方法

| 方法 | 签名 | 说明 |
|------|------|------|
| `studio_local(build_id?)` | `(build_id?) → String` | 返回 `host:port/app?build=N` 本地地址 |
| `studio_local_host()` | `() → String` | 返回 `host:port` 本地地址（无路径） |
| `studio_ext(build_id?)` | `(build_id?) → String` | 返回外部可访问的地址 |
| `studio_ext_host()` | `() → String` | 返回外部地址 host:port |
| `run(env, cmd, args)` | `(map, string, [string]) → nil` | 触发子进程运行请求 |
| `set_run_items(items)` | `([object]) → nil` | 注册可运行项列表 |

`set_run_items` 每个 item 包含：
- `name: string` — 唯一名称
- `in_studio: bool` — 是否在 Studio 中运行
- `on_run: function(build_id)` — 运行回调

## 类型转换辅助函数

| 函数 | 说明 |
|------|------|
| `script_value_to_string(vm, value)` | 将 ScriptValue 转为 String |
| `script_value_to_checked_string(vm, value, what)` | 同上，但检查 `.is_err()` |
| `script_value_to_query_id(vm, value, what)` | 解析 build_id（数字或 nil） |
| `script_value_to_bool(value)` | 解析布尔值（支持 bool 和 number ≠ 0） |
| `script_value_to_string_array(vm, value, what)` | 解析字符串数组 |
| `script_value_to_string_map(vm, value, what)` | 解析字符串映射（object） |
| `parse_registered_run_item(vm, value)` | 解析 RunItem 对象 |

## 地址辅助函数

| 函数 | 说明 |
|------|------|
| `normalize_studio_host(base)` | 规范化 hub 地址（去 scheme/路径） |
| `studio_url_for_app(base, build_id)` | 生成 app 连接 URL |

## 测试 (`#[cfg(test)]`)

| 测试 | 验证点 |
|------|--------|
| `studio_url_for_app_appends_app_path_to_base_addr` | `127.0.0.1:8001` + build_id=7 → `127.0.0.1:8001/app?build=7` |
| `studio_url_for_app_normalizes_existing_app_url` | `http://127.0.0.1:8001/app/5` → `127.0.0.1:8001/app?build=9` |

## 设计观察

- 每个挂载点最多一个运行中的脚本
- `ScriptHost` 通过 `vm.host.downcast_ref` 从脚本 VM 内部访问
- 脚本 API 使用 `macro_rules` 风格的 `script_args_def!` 和 `script_value!` 宏
- RunItem 注册在脚本层面，Cargo 构建信息在 Rust 层面，两者通过 `ScriptCommand::RunItem` 桥接
- `run()` 不直接执行命令，而是发送 `HubEvent::ScriptRunRequest` 让 dispatch 层处理
