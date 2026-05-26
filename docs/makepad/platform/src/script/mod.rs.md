# `platform/src/script/mod.rs` — 脚本集成模块入口

## 文件职责

该文件是 Makepad 平台层与 Splash VM 脚本系统集成的**模块根**。它声明了所有子模块并提供了统一的 `script_mod()` 入口函数，负责按照正确的依赖顺序注册所有平台脚本功能。

---

## 模块声明

- `cx`：注册 `cx` 模块，暴露 `cx.quit()`、`cx.os_type()` 等平台上下文方法
- `draw`：注册 `draw` 模块，暴露 `DrawCallUniforms`、`MouseCursor`、`ScriptDrawPass` 等绘制/窗口类型
- `event`：注册 `KeyCode` 枚举到 `draw` 模块下
- `res`：注册 `res` 模块，提供文件/HTTP/二进制资源加载能力
- `script`：定义 `CxScriptData` 结构体，捆绑脚本运行时的全部状态
- `std`：提供 `Cx` 上的 `with_vm()`/`eval()` 等桥接方法，委托给 `makepad_script_std`
- `timer`：注册 `std.random`、`std.start_timeout`、`std.start_interval` 等定时器/RNG 方法
- `vm`：定义 `ScriptVmCx` trait，实现 `ScriptVm` ↔ `Cx` 的双向指针交换

`pub use self::std::{fs, net, run}` 将 `makepad_script_std` 的 `fs`/`net`/`run` 模块提升到本模块的公共 API。

---

## `pub fn script_mod(vm: &mut ScriptVm)`

这是平台层脚本集成的**唯一初始化入口**，按照依赖顺序依次注册各模块：

1. `crate::script::cx::script_mod(vm)` — 先注册 `cx` 模块
2. `makepad_script_std::script_mod(vm)` — 再注册标准库（fs/net/WebSocket/task 等）
3. `crate::script::timer::script_mod(vm)` — 定时器和 RNG
4. `crate::script::res::script_mod(vm)` — 资源加载
5. `crate::script::draw::script_mod(vm)` — 绘制类型和窗口句柄
6. `crate::script::event::script_mod(vm)` — 事件类型（KeyCode）

**注意**：`script` 和 `std` 子模块没有单独的 `script_mod()` 函数——前者只是数据结构定义，后者直接在 `Cx` 上实现方法。模块的初始化顺序保证了在注册 UI 绘制的 `draw` 模块之前，资源加载、定时器、标准库均已就绪。
