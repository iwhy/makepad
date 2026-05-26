# `platform/src/script/timer.rs` — 定时器与随机数生成

## 文件职责

该文件实现了 Splash VM 脚本的定时器系统和伪随机数生成器。它在脚本的 `std` 模块上注册了 `random`、`random_seed`、`random_u32`、`start_timeout`、`start_interval`、`stop_timer` 方法，使脚本能够生成随机数和调度延迟/重复执行的回调。

---

## 核心数据结构

### `CxScriptTimer`

单个定时器条目：
- `id: LiveId` — 该定时器的唯一标识符，用于 `stop_timer` 查找
- `repeat: bool` — `true` 表示 setInterval（重复），`false` 表示 setTimeout（一次性）
- `timer: Timer` — 底层平台定时器句柄（由 `Cx::start_timeout` / `Cx::start_interval` 创建）
- `callback: ScriptFnRef` — 定时器触发时要调用的脚本函数引用

### `CxScriptTimers`

定时器管理器，包装一个 `Vec<CxScriptTimer>`：
```rust
#[derive(Clone, Default)]
pub struct CxScriptTimers {
    pub timers: Vec<CxScriptTimer>,
}
```

---

## `Cx::handle_script_timer(event)`

当平台定时器事件到来时调用。处理逻辑：

1. **匹配定时器**：遍历所有注册的 `CxScriptTimer`，通过 `timer.is_timer(event)` 匹配触发的底层平台定时器
2. **判断是否移除**：如果是非重复定时器（`setTimeout`），执行前从列表中移除
3. **传递时间参数**：如果事件携带了精确时间信息，将其转换为脚本值作为回调参数；否则传递 `NIL`
4. **调用回调**：通过 `self.with_vm_and_async(|vm| { vm.call(callback, &[time]) })` 在脚本 VM 中执行回调函数。使用 `with_vm_and_async` 是因为回调可能包含 `await` 表达式

---

## 脚本 API 注册 (`pub fn script_mod`)

### 句柄类型

```rust
vm.new_handle_type(id_lut!(timer));
```

注册 `timer` 句柄类型，预留扩展使用。

### 随机数生成 (RNG)

#### `next_hash(bytes)`

一个简单的 64 位哈希函数，基于 `0xd6e8_feb8_6659_fd93` 乘数和 XOR-shift 算法：
1. 累加输入字节
2. 进行三次 `XOR-shift → 乘常数` 轮次
3. 返回最终的哈希值

#### `fresh_seed()`

基于当前系统时间（纳秒）和进程 ID 生成初始化种子：
1. 获取 `SystemTime::now()` 的纳秒数
2. 将高 64 位和低 64 位异或作为种子
3. 混入 `std::process::id()`
4. 如果结果为零，使用一个固定默认值作为后备

#### `ensure_seeded(cx)`

检查 `cx.script_data.random_seed` 是否为零，为零则调用 `fresh_seed()` 初始化。

#### `std.random_seed()`

- 强制重新初始化随机数种子
- 调用 `fresh_seed()` 替换当前种子
- 返回 `NIL`

#### `std.random()` → `f64`

生成一个 `[0, 1)` 范围的伪随机浮点数：
1. 确保已播种
2. 对当前种子调用 `next_hash` 计算下一个状态
3. 更新种子
4. 将哈希值除以 `u64::MAX` 归一化到 `[0, 1)` 区间

#### `std.random_u32()` → `f64`

生成一个 `[0, 2^32)` 范围的伪随机浮点数：
1. 与 `random()` 相同的三步流程
2. 将哈希值截断为 `u32` 后再转为 `f64`
3. 返回值在 `[0, 2^32)` 范围内（注意返回类型是 `f64` 而非整数）

### 定时器方法

#### `std.start_timeout(delay, callback)` → `LiveId`

创建一次性定时器（类比 JavaScript 的 `setTimeout`）：
1. 验证参数：`delay` 必须是数字，`callback` 必须是函数
2. 通过 `ScriptFnRef::script_from_value(vm, callback)` 创建函数引用
3. 调用 `cx.start_timeout(delay)` 创建底层平台定时器（延迟以秒为单位）
4. 生成唯一的 `LiveId`
5. 构造 `CxScriptTimer` 条目（`repeat: false`）并推入 `timers` 列表
6. 返回定时器的 `LiveId`（可用于 `stop_timer`）

#### `std.start_interval(delay, callback)` → `LiveId`

创建重复定时器（类比 JavaScript 的 `setInterval`）：
1. 与 `start_timeout` 逻辑相同，但 `repeat` 设为 `true`
2. 使用 `ScriptFnRef::script_type_check` 替代 `vm.bx.heap.is_fn` 检查函数类型
3. 调用 `cx.start_interval(delay)` 创建底层重复定时器

#### `std.stop_timer(timer)` → `NIL`

停止并移除定时器：
1. 验证参数类型是否为 `LiveId`
2. 通过 `cx.script_data.timers.timers.retain(|v| v.id != timer)` 从列表中移除匹配的条目
3. 返回 `NIL`
