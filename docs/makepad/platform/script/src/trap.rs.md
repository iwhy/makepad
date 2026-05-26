# `trap.rs` 源码解读

**路径:** `platform/script/src/trap.rs`
**行数:** 98
**核心职责:** 实现脚本引擎的错误捕获与传播机制（`ScriptTrap`），以及使用 `script_err_gen!` 宏声明 18 种标准化的错误类别。

---

## 结构体 `ScriptError`

**路径:** 第7-12行

单个错误信息的容器：
- `message: String`：错误描述
- `origin_file: String`：错误发生的源文件名
- `origin_line: u32`：错误发生的行号
- `value: ScriptValue`：关联的脚本值（用于在错误上下文中保留值）

---

## 枚举 `ScriptTrapOn`

**路径:** 第15-19行

错误捕获后的处理策略：
- `Pause`：暂停执行
- `Return(ScriptValue)`：返回一个指定值
- `Bail(ScriptValue)`：中止并携带一个值

---

## 结构体 `ScriptTrapInner`

**路径:** 第21-26行

内部错误捕获状态：
- `err: RefCell<VecDeque<ScriptError>>`：错误队列（可同时存储多个错误）
- `on: Cell<Option<ScriptTrapOn>>`：当前捕获策略
- `ip: ScriptIp`：执行指令指针

---

## 枚举 `ScriptTrap<'a>`

**路径:** 第28-32行

错误捕获句柄（轻量级枚举，避免在无错误路径上的堆分配）：
- `NoTrap`：无捕获（执行上下文中忽略错误）
- `Inner(&'a ScriptTrapInner)`：挂载到实际的错误状态

---

## impl ScriptTrap

### `pass(self)`（第37-39行）

透传自身，用于所有权转换。

---

## impl ScriptTrapInner

### `pass(&'a self)`（第43-46行）

返回包装为 Inner 的捕获句柄。

### `push_err(value, message, origin_file, origin_line)`（第49-63行）

将新错误压入队列并返回传入的 value（链式调用友好）。

### `ip(&self)`（第64-66行）

获取当前指令指针位置。

### `goto(wh)`（第67-69行）

跳转到指定指令地址。

### `goto_rel(wh)`（第70-72行）

相对跳转。

### `goto_next()`（第74-77行）

前进到下一条指令（ip.index += 1）。

---

## 错误宏声明

**路径:** 第79-98行

使用 `script_err_gen!` 宏声明了 18 种错误类别（从原来的 56 个精简合并）：

| 宏名称 | 用途 |
|--------|------|
| `script_err_not_found` | 查找失败 |
| `script_err_type_mismatch` | 类型不匹配 |
| `script_err_wrong_value` | 期望不同的值 |
| `script_err_out_of_bounds` | 索引/越界错误 |
| `script_err_immutable` | 不可修改 |
| `script_err_stack` | 栈错误 |
| `script_err_invalid_args` | 参数错误 |
| `script_err_not_allowed` | 操作不允许 |
| `script_err_inconsistent` | 跨分支类型不一致 |
| `script_err_not_impl` | 未实现 |
| `script_err_unexpected` | 兜底错误 |
| `script_err_assert_fail` | 断言失败 |
| `script_err_user` | 用户生成的错误 |
| `script_err_pod` | 所有 pod 错误 |
| `script_err_shader` | 所有着色器错误 |
| `script_err_unknown_type` | 类型未注册 |
| `script_err_duplicate` | 键已存在 |
| `script_err_io` | 文件系统/子进程 I/O 错误 |
| `script_err_limit` | 资源限制 |
