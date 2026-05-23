# `thread.rs` — 执行线程系统：`ScriptThread` 与线程容器

## 文件位置
`platform/script/src/thread.rs` (403 行)

## 核心类型

### `StackBases` — 栈基座快照
```rust
pub struct StackBases {
    pub loops: usize,   // 循环帧栈深度
    pub tries: usize,   // try 帧栈深度
    pub stack: usize,   // 操作数栈深度
    pub scope: usize,   // 作用域链深度
    pub mes: usize,     // 消息/对象栈深度
}
```
这是一个 **检查点结构**：在函数调用或循环开始前记录各栈的当前深度，返回时截断到该深度以恢复执行上下文。

---

### `LoopValues` — 循环变量
```rust
pub struct LoopValues {
    pub value_id: LiveId,       // 值标识符（迭代变量，如 `item`）
    pub key_id: Option<LiveId>,     // 键标识符（可选，如 `index`）
    pub index_id: Option<LiveId>,   // 索引标识符（可选，如 `idx`）
    pub source: ScriptValue,    // 被迭代的源对象/数组
    pub index: f64,             // 当前迭代位置
}
```
支持 **多重解构迭代**：`for (value, key, index) in source` 三种标识符可选绑定。

---

### `TryFrame` — 异常处理帧
```rust
pub struct TryFrame {
    pub push_nil: bool,   // 进入 try 时是否需要压入 NIL（用于 finally 块）
    pub start_ip: u32,    // try 块起始指令地址
    pub jump: u32,        // catch/finally 块跳转地址偏移
    pub bases: StackBases,// 进入 try 时的栈基座快照
}
```

### `LoopFrame` — 循环帧
```rust
pub struct LoopFrame {
    pub values: Option<LoopValues>, // 循环变量
    pub start_ip: u32,              // 循环体起始指令地址
    pub jump: u32,                  // 循环结束跳转地址偏移
    pub bases: StackBases,          // 进入循环时的栈基座快照
}
```
`start_ip` 用于 `continue` 指令跳回，`jump` 用于 `break` 指令跳出。

---

### `CallFrame` — 调用帧
```rust
pub struct CallFrame {
    pub bases: StackBases,          // 调用前的栈基座
    pub args: OpcodeArgs,           // 参数编码信息
    pub return_ip: Option<ScriptIp>,// 返回地址
}
```
`return_ip` 为 `None` 时表示这是最外层调用（没有返回点）。

---

### `ScriptMe` — 消息/对象栈条目
```rust
pub enum ScriptMe {
    Object(ScriptObject),      // 普通对象引用
    Call {
        sself: Option<ScriptValue>, // self 值（可选，方法调用时指向接收者）
        args: ScriptObject,         // 参数对象
        method: Option<LiveId>,     // 方法名（动态分发时使用）
    },
    Pod {
        pod: ScriptPod,            // POD 数据
        offset: ScriptPodOffset,   // 字段偏移
    },
    Array(ScriptArray),         // 数组引用
}
```
`ScriptMe` 存储的是 **正在遍历或正在操作的"当前上下文"**。每种变体对应不同的遍历/访问模式：
- `Object`：正在遍历一个对象的属性。
- `Call`：正在构建一个函数调用（self/参数）。
- `Pod`：正在访问一个 POD 的字段。
- `Array`：正在遍历一个数组元素。

`Into<ScriptValue>` 实现将各变体统一转换为 `ScriptValue`，`Call` 变体转换时忽略 `sself` 和 `method`，只保留参数对象。

---

### `ScriptThreadId(u32)` — 线程标识符
简单封装 `u32`，提供 `to_index()` 方法转为 `usize` 用于数组索引。

---

### `ScriptThread` — 执行线程
```rust
pub struct ScriptThread {
    pub(crate) is_paused: bool,          // 暂停标志
    pub(crate) stack_limit: usize,       // 栈深度上限（默认 1,000,000）
    pub(crate) tries: Vec<TryFrame>,     // try 帧栈
    pub(crate) loops: Vec<LoopFrame>,    // 循环帧栈
    pub(crate) scopes: Vec<ScriptObject>,// 作用域链
    pub(crate) stack: Vec<ScriptValue>,  // 操作数栈
    pub(crate) calls: Vec<CallFrame>,    // 调用栈
    pub(crate) mes: Vec<ScriptMe>,       // 消息对象栈
    pub trap: ScriptTrapInner,           // 陷阱/错误处理
    pub(crate) json_parser: JsonParserThread,// 线程局部的 JSON 解析器
    pub(crate) thread_id: ScriptThreadId,// 线程 ID
}
```

#### 五栈架构
Makepad 脚本引擎采用**分离栈**设计，而非单一栈：

| 栈 | 用途 | 操作 |
|----|------|------|
| `stack` | 操作数栈 — 计算表达式、传递参数 | push/pop/resolve |
| `calls` | 调用栈 — 函数调用跟踪 | push/pop 帧 |
| `scopes` | 作用域链 — 变量查找 | push/pop scope |
| `loops` | 循环帧栈 — break/continue | push/pop 帧 |
| `tries` | 异常处理栈 — try/catch | push/pop 帧 |
| `mes` | 消息对象栈 — 遍历/调用上下文 | push/pop 帧 |

这种分离设计使得各栈可以独立管理，`truncate_bases` 可以一次性截断所有栈到基线。

---

## `ScriptThread` 方法

### `new(thread_id)` — 创建线程
初始化所有的栈为空，设置默认 `stack_limit` 为 1,000,000。

### `new_bases()` — 创建当前状态快照
记录所有六个栈的当前深度为 `StackBases`。

### `pause()` / `is_paused()` — 暂停控制
`pause()` 设置 `trap.on` 为 `Pause` 并标记 `is_paused`，返回线程 ID（可用于恢复）。
`is_paused()` 检查 `is_paused` 布尔值。

### `thread_id()` — 返回线程 ID

### `truncate_bases(bases, heap)` — 截断到基座
实现逻辑：
1. 对 `tries`、`loops`、`stack`、`mes` 直接调用 `truncate(bases.xxx)`。
2. 对 `scopes` 调用 `free_unreffed_scopes` 逐级弹出并释放未引用的作用域对象。
3. 这是函数返回/异常展开的核心清理操作。

### `free_unreffed_scopes(bases, heap)` — 释放未引用作用域
实现逻辑：
1. `while self.scopes.len() > bases.scope` 逐个 `pop` 作用域对象。
2. 每个 popped 的作用域调用 `heap.free_object_if_unreffed(scope)`，尝试释放（如果 GC 引用计数归零）。
3. 注释说明此功能实际上在特定场景下被禁用（"DISABLED: investigating RootObject already freed"），说明这是一个历史遗留问题。

### `pop_stack_resolved(heap)` — 弹出并解析值
关键方法，实现**延迟标识符解析**：
1. `self.stack.pop()` 弹出值。
2. 如果是 `id`（`as_id()` 返回 `Some` 且**非逃逸 ID**），调用 `self.scope_value(heap, id)` 解析到作用域中的实际值。
3. 如果是逃逸 ID，直接返回（逃逸 ID 是已经显式标记的值，不再解析）。
4. 否则直接返回普通值。
5. 空栈时触发栈错误。

**设计动机**：脚本编译时，局部变量引用被编译为 `LiveId` 压栈。访问时不需要每次都查作用域——只在执行时按需解析，且解析后如果是值类型直接使用，引用类型则继续追踪。

### `peek_stack_resolved(heap)` — 窥顶并解析
与 `pop_stack_resolved` 逻辑相同，但不弹出（使用 `last()` 而非 `pop()`）。

### `peek_stack_value()` — 窥顶（不解析）
直接返回栈顶值，不解析 ID。用于只关心值类型而不需要跟踪的场景。

### `peek_stack_value_at(offset)` — 按偏移量窥视
从栈顶向下偏移 `offset` 位置取值，不弹出。`offset = 0` 等价于 `peek_stack_value`。

### `pop_stack_value()` — 弹出（不解析）
直接 `pop()`，不触发 ID 解析。用于已经解析过的值或不需要解析的值。

### `push_stack_value(value)` — 压栈（带上限检查）
1. 检查 `self.stack.len() > self.stack_limit`。
2. 如果超限触发栈溢出错误。
3. 否则 `self.stack.push(value)`。

### `push_stack_unchecked(value)` — 无检查压栈
直接 `push`，无上限检查。用于内部的、已知安全的压栈操作。

### `call_has_me()` — 检查当前调用是否有消息对象
比较当前调用帧记录的 `mes` 基座与当前 `mes.len()`。如果当前 `mes` 深度超过了调用时的深度，说明调用栈中有待处理的消息对象。

### `call_has_try()` — 检查当前调用是否有 try 帧
类似 `call_has_me`，但检查 try 帧。如果当前 try 深度超过调用时的深度，说明有活跃的 try/catch 块。

### `scope_value(heap, id)` — 作用域变量查找
实现逻辑：
1. 获取作用域链顶部：`*self.scopes.last().unwrap()`。
2. 委托 `heap.scope_value(scope, id.into(), trap)` 沿着作用域链查找变量。
3. 这个调用可能沿着 `proto` 链向上查找。

### `set_scope_value(heap, id, value)` — 设置作用域变量
1. 获取作用域链顶部。
2. 委托 `heap.set_scope_value(scope, id.into(), value, trap)` 从顶部向下查找并设置。

### `def_scope_value(heap, id, value)` — 定义作用域变量（可能 shadow）
实现逻辑：
1. 委托 `heap.def_scope_value(scope, id, value)`。
2. 如果返回 `Some(new_scope)`，说明变量被定义在新的 shadow 作用域中，将 new_scope 推入作用域链。
3. 这是 `var/let` 声明的核心实现——在需要 shadow 时创建新作用域层。

---

## `ScriptThreads` — 线程容器

### 设计动机
`ScriptThreads` 不是简单的 `Vec<ScriptThread>`，它通过**缓存原始指针**解决了同时借用 `ScriptThreads` 和 `ScriptHeap` 的问题。

### 核心字段
```rust
pub struct ScriptThreads {
    threads: Vec<ScriptThread>,
    current: usize,
    cur_ptr: *mut ScriptThread,  // 指向当前线程的缓存指针
}
```

### `new()` — 创建含一个默认线程的容器
创建 `ScriptThreadId(0)` 的线程，并初始化 `cur_ptr`。

### `empty()` — 创建空容器
线程数为 0，`cur_ptr` 为 null。

### `update_ptr()` — 刷新缓存指针
内部方法，在 `current` 变化或 `threads` 重新分配后调用。

### `cur()` — 获取当前线程（可变引用）
通过 `unsafe { &mut *self.cur_ptr }` 直接从缓存指针读取。需要 `debug_assert!` 检查指针非空。

### `trap()` — 获取当前线程的陷阱
`unsafe { (*self.cur_ptr).trap.pass() }`。

### `cur_ref()` — 获取当前线程（不可变引用）
类似 `cur()` 但返回 `&ScriptThread`。

### `set_current(id)` — 切换当前线程
设置 `current` 并刷新缓存指针。

### `current()` — 返回当前线程索引

### `set_current_to_first_unpaused_thread()` — 查找第一个非暂停线程
实现逻辑：
1. 遍历 `threads`，找到第一个 `!thread.is_paused` 的线程并设置为当前线程。
2. 如果没有非暂停线程，创建一个新线程（ID 递增）。
3. 这种设计支持**协程式切换**：暂停的线程可以随后恢复。

### `set_current_thread_id(thread_id)` — 按 `ScriptThreadId` 切换
通过 `thread_id.to_index()` 设置 `current`。

### `len()` / `is_empty()` — 容器大小查询

### `push(thread)` — 追加线程
调用后刷新指针，因为 `Vec::push` 可能触发重新分配。

### `get(index)` / `get_mut(index)` — 按索引访问
标准 `Vec` 委托。
