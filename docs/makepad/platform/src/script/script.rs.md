# `platform/src/script/script.rs` — 脚本运行时数据捆绑

## 文件职责

该文件定义了 `CxScriptData` 结构体，这是整个 Makepad 平台层脚本运行时的**单一数据聚合容器**。它被嵌入 `Cx` 结构体（平台上下文）中，将脚本引擎所需的所有运行时状态捆绑在一个结构体内供 `Cx` 统一管理。

---

## `CxScriptData`

```rust
#[derive(Default)]
pub struct CxScriptData {
    pub std: ScriptStd,
    pub random_seed: u64,
    pub timers: CxScriptTimers,
    pub resources: CxScriptResources,
    pub crate_manifests: Rc<RefCell<HashMap<String, String>>>,
    pub live_reload: CxLiveReloadState,
}
```

### 各字段说明

#### `std: ScriptStd`

来自 `makepad_script_std` 的标准库状态，包含：
- `ScriptTaskScheduler`：异步任务（网络请求、文件 I/O）的调度器
- 线程池管理（WebSocket、HTTP 请求队列）
- `with_vm`/`pump` 功能的中间状态

#### `random_seed: u64`

伪随机数生成器的当前种子值。初始为 0，首次调用 `random()` 或 `random_u32()` 时通过 `fresh_seed()` 初始化。每次生成随机数后更新种子，产生下一伪随机数。

#### `timers: CxScriptTimers`

定时器管理器，定义在 `timer.rs` 中。维护一个 `Vec<CxScriptTimer>`，每个条目包含：
- `LiveId`：唯一 ID
- `repeat: bool`：true 表示 `setInterval`，false 表示 `setTimeout`
- `Timer`：底层平台定时器句柄
- `ScriptFnRef`：定时器触发时调用的脚本回调函数

#### `resources: CxScriptResources`

资源加载管理器，定义在 `res.rs` 中。包含：
- `Vec<CxScriptResource>`：所有资源条目列表
- `HashMap<String, ScriptHandle>`：绝对路径→句柄的快速查找映射
- `Vec<CxScriptHttpResource>`：进行中的 HTTP 请求列表

#### `crate_manifests: Rc<RefCell<HashMap<String, String>>>`

Crate 清单路径映射表。键为 crate 名称（如 `"makepad_widgets"`），值为该 crate 的 `Cargo.toml` 所在目录的绝对路径。

使用 `Rc<RefCell<>>` 是因为：
- 在 `Cx` 内部和 `ScriptVm` 外部都可能需要访问这个映射
- 脚本评估期间 `ScriptVm` 的 `bx` 被暂时交换到 `Cx` 上，但其 `code.crate_manifests` 映射仍需要可访问
- 这个共享引用使得 `res.rs` 中的 `resolve_crate_resource_paths` 可以随时读取 manifests，无论当前 `ScriptVm` 处于何种状态

#### `live_reload: CxLiveReloadState`

热重载状态管理，来自 `crate::live_reload`。在开发模式下监控文件变化，触发脚本模块的热更新。

---

## 设计意图

`CxScriptData` 将所有脚本相关状态从 `Cx` 结构中逻辑分离出来，但又保持物理上在 `Cx` 内部。这种设计带来的好处：

1. **统一聚合**：所有脚本运行时状态在一个地方定义，`Cx` 不必分散管理多个零散字段
2. **单一 `Default`**：通过 `#[derive(Default)]` 一次性初始化所有脚本状态
3. **隔离性**：平台层代码可以直接访问 `Cx` 的非脚本字段，而脚本集成代码则通过 `CxScriptData` 访问其所需状态
4. **共享指针**：对于需要跨 `Cx` 和 `ScriptVm` 边界的字段（如 `crate_manifests`），使用 `Rc<RefCell<>>` 模式实现共享可变访问
