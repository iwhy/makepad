# `build_manager.rs` — 构建进程管理器

## 文件位置
- 路径: `studio/hub/src/build_manager.rs`
- 行数: 793 行
- 作用: 管理构建进程（cargo 或其他命令）的生命周期，包括进程组控制、输出捕获、环境变量注入

## 进程组管理 (`process_group` 模块)

### Windows 实现
```rust
#[link(name = "kernel32")]
extern "system" {
    fn CreateJobObjectW(...) -> *mut u8;
    fn AssignProcessToJobObject(...) -> i32;
    fn TerminateJobObject(...) -> i32;
    fn CloseHandle(...) -> i32;
}
```
使用 Windows Job Object API：
- `JobHandle::new()` — 创建 job object
- `assign(&child)` — 将子进程分配到 job
- `terminate()` — 终止整个 job 中的所有进程
- `Drop` 时 `CloseHandle`

### Unix 实现
```rust
pub fn configure_command(cmd: &mut Command) {
    cmd.pre_exec(|| { setpgid(0, 0); Ok(()) });
}
```
使用进程组（`setpgid` 创建新组）：
- `JobHandle::new()` → 空结构体
- `assign(&child)` → 记录 PID
- `terminate()` → `kill(-pid, SIGKILL)` 杀死整个进程组

## 核心数据结构

### `RunningBuild`
```rust
struct RunningBuild {
    info: BuildInfo,
    child: RunningChild,
}
```

### `RunningChild`
```rust
struct RunningChild {
    child: Arc<Mutex<Child>>,
    job: process_group::JobHandle,
}
```
- `Arc<Mutex<Child>>` 供输出读取线程和等待线程共享
- `terminate()`: 先尝试 `job.terminate()`（杀整个进程组），失败则 fallback `child.kill()`
- `send_stdin(text)`: 锁定 child → 获取 stdin → 写入并 flush

## `BuildManager`

```rust
pub struct BuildManager {
    builds: HashMap<QueryId, RunningBuild>,
}
```

### 核心方法

| 方法 | 说明 |
|------|------|
| `start_command_run(build_id, mount, package, cwd, program, args, env, inject_studio_env, studio_addr, event_tx)` | 启动任意命令进程 |
| `start_cargo_run(build_id, mount, cwd, args, env, studio_addr, event_tx)` | 启动 cargo run（Unix 下可能使用 direct stdio 模式） |
| `stop_build(build_id)` | 终止构建进程 |
| `mark_exited(build_id, exit_code)` | 标记构建已退出，返回 `BuildInfo` |
| `send_stdin(build_id, text)` | 向构建进程 stdin 写入 |
| `list_builds() -> Vec<BuildInfo>` | 按 build_id 排序列出所有构建 |
| `package_for_build(build_id) -> Option<&str>` | 获取构建所属包名 |

### `start_command_run` 流程

1. 检查 `build_id` 是否已存在（防止重复）
2. 配置命令：`process_group::configure_command` → `stdin/stdout/stderr piped`
3. 环境变量注入：
   - 强制设置 `RUST_BACKTRACE=1`
   - 解析 `STUDIO_HOST`：优先 `STUDIO_HOST` 环境变量，其次 `STUDIO` URL 中解析，最后使用 `studio_addr` 参数
   - 设置 `STUDIO_BUILD`：默认 `build_id.0.to_string()`
   - 设置 `STUDIO_CRATE`：默认 `package`
   - 移除 `STUDIO`（规范化后不再需要）
4. spawn 命令 → 分离 stdout/stderr 读取线程 → 分离 wait 线程
5. 存入 `HashMap<QueryId, RunningBuild>`

### `start_cargo_run`

- 默认调用 `start_command_run` 执行 `cargo` 命令
- Unix 下特殊优化：`should_use_direct_stdio_run` 检查条件：`MAKEPAD=headless` + `cargo run` + `--stdin-loop`
- Direct stdio 模式将 `cargo run` 拆分为 `cargo build + exec ./binary`，减少重定向层数

### 环境变量解析辅助函数

| 函数 | 说明 |
|------|------|
| `normalize_studio_host(base)` | 标准化 URL → `host:port`，去除 scheme、路径、query |
| `studio_query_value(studio, key)` | 从 `STUDIO` URL 的 query 中提取参数 |
| `extract_studio_build_id(studio)` | 提取 build ID（query 或 path 格式） |
| `extract_studio_crate_name(studio)` | 提取 crate 名称（query 参数） |

## 输出和生命周期管理

### `spawn_reader(build_id, is_stderr, reader, event_tx)`
- 在独立线程中逐行读取 stdout/stderr
- stdout 的行首先尝试反序列化为 `AppToStudio` JSON 消息 → 成功则发送 `ProcessAppMessage`
- 否则发送 `ProcessOutput { build_id, is_stderr, line }`

### `spawn_waiter(build_id, child, event_tx)`
- 循环以 30ms 间隔调用 `child.try_wait()`
- 进程退出后发送 `ProcessExited { build_id, exit_code }`

## Cargo Manifest 解析

### `CargoManifestTargets`
```rust
struct CargoManifestTargets {
    package_name: Option<String>,
    bin_names: Vec<String>,
}
```

`default_binary_name()` 策略：
- 无 bin targets → `package_name`
- 一个 bin target → 该 bin 名
- 多个 bin targets → 仅当 `package_name` 匹配某个 bin 名时返回它，否则 None

### 解析函数

| 函数 | 说明 |
|------|------|
| `parse_manifest_targets(manifest)` | 解析 `Cargo.toml` 的 `[package]` 和 `[[bin]]` 节 |
| `parse_manifest_string_value(line, key)` | 解析 TOML `key = "value"` |
| `read_manifest_targets(cwd)` | 读取工作目录的 Cargo.toml |
| `parse_package_name(args)` | 从 cargo args 中提取包名（`-p` / `--package` / `--bin`） |
| `parse_cargo_flag_value(args, flags)` | 解析 cargo 命令行标志值 |

### Direct Stdio Run (Unix)

`build_direct_stdio_run_script(cwd, args, env) -> Option<String>`：
1. 查找 `--` 分隔符
2. 解析二进制名称
3. 构造 `cargo build ... && exec ./target/release/{binary} {app_args}`
4. 参数做 shell escape（单引号包裹，内部 `'` 转义为 `'"'"'`）

## 测试 (`#[cfg(test)]`)

| 测试 | 验证点 |
|------|--------|
| `parse_manifest_targets_reads_single_bin_name` | Cargo.toml 解析正确，`default_binary_name` 返回单 bin |
| `build_direct_stdio_run_script_uses_resolved_bin_name` | direct stdio 脚本包含正确的 build 命令和 binary 路径 |

## 设计观察

- 进程组管理跨平台：Win32 Job Object vs Unix process group (setpgid)
- stdout 解析 `AppToStudio` JSON 消息实现 app → hub 通信（无需额外 WebSocket）
- 环境变量 `STUDIO`/`STUDIO_HOST`/`STUDIO_BUILD`/`STUDIO_CRATE` 形成完整的子进程回连协议
- spawn_reader 和 spawn_waiter 使用独立线程而非 async，避免与 hub 主循环耦合
