# vm.rs — ScriptVm: Splash 字节码虚拟机

## 概述

`vm.rs` 定义了 Splash VM 的核心结构 `ScriptVm`，包含解释器主循环 `run_core()`、模块管理、函数调用、增量求值（流式 REPL）、错误处理、句柄类型注册等全部运行时基础设施。

**总行数**: 1250 行

---

## 核心数据结构

### `ScriptVm<'a>` (行 165-1153)

```rust
pub struct ScriptVm<'a> {
    pub host: &'a mut dyn Any,   // 宿主应用状态
    pub std: &'a mut dyn Any,    // 标准库状态
    pub bx: Box<ScriptVmBase>,   // 堆分配的 VM 基座
}
```

**字段说明**:
- `host`: 宿主应用（如 Makepad Studio）的状态引用，通过 `dyn Any` 擦除类型后由宿主在运行时向下转换
- `std`: 标准库的状态引用（如时间、文件系统、控制台等的内部状态），同样擦除类型
- `bx`: `Box<ScriptVmBase>` — 所有 VM 可变状态的堆分配容器

**设计理由**: `ScriptVm` 需要持有对外部状态的引用，同时又要在内部存储大量可变 VM 状态。`host` 和 `std` 借用外部，`bx` 是 VM 自身的所有权状态。

### `ScriptVmBase` (行 1155-1209)

```rust
pub struct ScriptVmBase {
    pub void: usize,
    pub code: ScriptCode,
    pub heap: ScriptHeap,
    pub threads: ScriptThreads,
    pub injected_globals: HashMap<LiveId, ScriptValue>,
    pub is_reload: bool,
    pub debug_trace: bool,
    pub silence_errors: bool,
}
```

**字段说明**:
- `void`: 保留字段，用于 void 值判定
- `code`: 编译后的所有脚本代码（`ScriptCode`）
- `heap`: 堆管理器（对象、数组、字符串、POD、正则等的存储）
- `threads`: 多线程调度器（`ScriptThreads`）
- `injected_globals`: 由宿主注入的全局变量（如 `__script_source__`）
- `is_reload`: 热重载标记
- `debug_trace`: 是否启用调试追踪
- `silence_errors`: 是否静默错误（增量求值时使用）

#### `ScriptVmBase::empty()` (行 1167-1178)

创建一个空的 VM 基座，所有字段取默认值/空值。用于测试场景。

#### `ScriptVmBase::new()` (行 1180-1209)

完整的 VM 初始化。执行以下步骤：
1. 创建空堆 `ScriptHeap::empty()`
2. 创建 `ScriptNative`，注册所有内建模块：
   - `define_math_module` — 数学函数（sin, cos, sqrt 等）
   - `define_std_module` — 标准库（print, type, len 等）
   - `define_regex_module` — 正则表达式
   - `define_html_module` — HTML 渲染
   - `define_shader_module` — GPU 着色器编译
   - `define_gc_module` — 垃圾回收控制
   - `define_pod_module` — POD 类型系统
3. 创建 `ScriptBuiltins`，缓存内建类型引用（如 `std.Range`）
4. 组装 `ScriptCode` 结构

### `ScriptCode` (行 88-95)

```rust
pub struct ScriptCode {
    pub builtins: ScriptBuiltins,
    pub native: RefCell<ScriptNative>,
    pub bodies: RefCell<Vec<ScriptBody>>,
    pub crate_manifests: Rc<RefCell<HashMap<String, String>>>,
    pub script_mod_overrides: Rc<RefCell<HashMap<ScriptModKey, String>>>,
}
```

**字段说明**:
- `builtins`: 缓存的内建类型引用（如 Range 对象），避免重复查找
- `native`: 原生函数注册表（`RefCell` 包裹以实现内部可变性）
- `bodies`: 所有已编译的脚本 body（每个 `script_mod!` 调用对应一个 body）
- `crate_manifests`: crate 名称到 Cargo.toml 路径的映射（用于 `crate_resource!` 路径解析）
- `script_mod_overrides`: 脚本模块代码覆盖表（热重载时可用）

### `ScriptBody` (行 59-68)

```rust
pub struct ScriptBody {
    pub source: ScriptSource,
    pub effective_code: String,
    pub tokenizer: ScriptTokenizer,
    pub parser: ScriptParser,
    pub scope: ScriptObjectRef,
    pub me: ScriptObjectRef,
    pub checkpoint: Option<ParserCheckpoint>,
    pub source_len: usize,
}
```

每个 `script_mod!` 调用编译为一个 `ScriptBody`，包含：
- `source`: 原始源代码（`ScriptMod` 或流式代码）
- `effective_code`: 实际执行代码（可能被覆盖）
- `tokenizer`/`parser`: 词法分析器和解析器（含操作码）
- `scope`/`me`: 作用域对象和 `self` 对象
- `checkpoint`: 解析器检查点（增量求值时使用）
- `source_len`: 已处理的源码长度

### `ScriptMod` (行 43-52)

```rust
pub struct ScriptMod {
    pub cargo_manifest_path: String,
    pub module_path: String,
    pub file: String,
    pub line: usize,
    pub column: usize,
    pub code: String,
    pub values: Vec<ScriptValue>,
}
```

描述一个 `script_mod!` 宏调用的编译单元，包含 Cargo 路径、文件位置、源代码和编译期常数值。

### `ScriptModKey` (行 27-41)

```rust
pub struct ScriptModKey {
    pub file: String,
    pub line: usize,
    pub column: usize,
}
```

用于在 `script_mod_overrides` 映射中唯一标识一个脚本模块。

### `ScriptBuiltins` (行 70-86)

```rust
pub struct ScriptBuiltins {
    pub range: ScriptObject,
    pub pod: ScriptPodBuiltins,
}
```

初始化时从堆中查找 `std.Range` 对象并缓存，避免每次使用 `Range` 时重复路径查找。

### `ScriptLoc` (行 97-113)

```rust
pub struct ScriptLoc {
    pub file: String,
    pub col: u32,
    pub line: u32,
}
```

源代码位置信息，Display 格式为 `file:line:col`。

---

## ScriptCode 方法

### `ip_to_loc(&self, ip: ScriptIp) -> Option<ScriptLoc>` (行 116-163)

将字节码 IP 地址映射回源代码位置。

**实现逻辑**:
1. 通过 `ip.body` 找到对应的 `ScriptBody`
2. 从 body 的 `parser.source_map` 中查找 `ip.index` 对应的 token
3. 如果直接映射为 None（合成操作码），则向左右两侧搜索最近的有效 token
4. 如果找到 token，通过 `tokenizer.token_index_to_row_col()` 获取行列号
5. 如果是 `ScriptMod` 源，加上模块的文件偏移（`script_mod.line`）
6. 如果所有查找失败，返回 `unknown` 位置

---

## ScriptVm 方法详解

### 基础访问器

| 方法 | 逻辑 |
|------|------|
| `heap()` | 返回 `&self.bx.heap` 的不可变引用 |
| `heap_mut()` | 返回 `&mut self.bx.heap` 的可变引用 |
| `println(value)` | 调用 `self.bx.heap.println(value.into())` 输出到 stdout |
| `thread()` | 返回当前线程 `&ScriptThread` 的不可变引用 |
| `thread_mut()` | 返回当前线程 `&mut ScriptThread` 的可变引用 |
| `trap()` | 返回当前线程的 `ScriptTrap` 追踪器 |
| `set_thread(id)` | 切换到指定 ID 的线程上下文 |
| `with_vm(f)` | 调用闭包 `f(self)` |
| `is_reload()` | 返回热重载标记 |
| `with_reload(f)` | 临时设置热重载标记为 true，执行闭包后恢复 |

### 错误格式化

| 方法 | 逻辑 |
|------|------|
| `format_enum_variant_error(value)` | 调用 `suggest::format_enum_variant_error` 生成枚举变体错误的描述 |
| `format_object_for_error(obj)` | 将对象格式化为错误消息：遍历原型链和键值，限制在 200 字符内 |

### 垃圾回收 (行 200-209)

| 方法 | 逻辑 |
|------|------|
| `gc()` | 执行 Mark & Sweep：先 mark 所有可达对象，再 sweep 清理。只记录 >1ms 的暂停 |
| `gc_with_status()` | 同上，但始终记录 GC 状态日志 |

### `bail(&mut self, msg: &str)` (行 176-184)

快速失败：在栈、作用域等意外为空时调用。创建一个 `script_err_unexpected!` 错误并设置 `trap.on = Some(Bail(err))`，使 `run_core` 在下一个循环迭代退出。

### `script_me_from_value(&mut self, me: ScriptValue) -> Option<ScriptMe>` (行 265-282)

将通用值转换为 `ScriptMe`（函数调用的 `self` 上下文）：
- nil → None
- object → `ScriptMe::Object(obj)`
- array → `ScriptMe::Array(arr)`
- pod → `ScriptMe::Pod { pod, offset: default }`

### `call_with_scope(&mut self, scope: ScriptObject, me: ScriptValue) -> ScriptValue` (行 284-328)

核心函数调用实现。

**实现逻辑**:
1. 通过 `heap.parent_as_fn(scope)` 查找作用域的父对象是否注册为函数
2. 如果是**原生函数**(`ScriptFnPtr::Native`)：
   - 从 `code.native` 中获取函数指针
   - 暂停当前线程（`is_paused = true`），允许原生函数内重入
   - `unsafe` 调用函数指针
   - 如果原生函数未设置 `Pause` trap，恢复线程
3. 如果是**脚本函数**(`ScriptFnPtr::Script`)：
   - 创建 `CallFrame`，保存当前栈基址
   - 推入作用域和 `me` 上下文
   - 设置跳转目标为脚本函数的起始 IP
   - 调用 `run_core()` 执行脚本
4. 如果既非原生也非脚本函数，返回错误

### `call(&mut self, fnobj, args) -> ScriptValue` (行 330-332)

最简函数调用，`me = NIL`。

### `call_with_self(&mut self, fnobj, args, sself) -> ScriptValue` (行 334-361)

调用函数并显式设置 `self` 参数。

**实现逻辑**:
1. 以 `fnobj` 为原型创建新作用域
2. 清除作用域的 deep 标记
3. 通过 `heap.push_all_fn_args` 将所有参数压入作用域
4. 如果 `sself` 非 nil，设置作用域的 `self` 键
5. 标记为 deep 对象和 auto 存储
6. 调用 `call_with_scope`

### `call_with_me(&mut self, fnobj, args, me) -> ScriptValue` (行 363-385)

调用函数并设置 `me`（消息接收者，类似 OOP 的 `this`）。

实现与 `call_with_self` 相同，但将 `me` 传递给 `call_with_scope`。

### `call_with_args_object(&mut self, fnobj, args_obj) -> ScriptValue` (行 387-393)

调用函数，参数以 `ScriptObject` 形式提供（无 `me`）。

### `call_with_args_object_with_me(&mut self, fnobj, args_obj, me) -> ScriptValue` (行 395-445)

最完整的参数传递方式。

**实现逻辑**:
1. 验证 `fnobj` 是对象类型
2. 以 `fnobj` 为原型创建新作用域
3. 设置存储模式为 `vec2`（支持命名和位置参数混合）
4. 遍历 `args_obj` 的 vec：
   - 位置参数（key 为 nil）→ 通过 `unnamed_fn_arg` 映射到命名参数
   - 命名参数（key 非 nil）→ 直接 `vec_push` 到作用域
5. 复制 map 条目（如 `self`、`ui`）到作用域
6. 调用 `call_with_scope`

### `drain_errors(&mut self)` (行 450-496)

清空错误队列并记录到日志。

**实现逻辑**:
1. 循环弹出错误队列
2. 如果 `silence_errors` 则跳过
3. 如果错误包含 IP，通过 `ip_to_loc` 获取位置，调用 `log_with_level` 记录
4. 如果没有位置信息，仍记录到日志

### `handle_errors(&mut self)` (行 500-520)

错误处理分发。

**实现逻辑**:
1. 检查当前调用帧是否有 try 块（`call_has_try()`）
2. 如果有 try 块：清空错误队列，弹出 try 帧，截断栈到 try 的基址，跳转到 catch 位置
3. 如果没有 try 块：调用 `drain_errors()` 记录所有错误

### `run_core(&mut self) -> ScriptValue` (行 522-593)

**解释器主循环**。这是 VM 最核心的方法。

**实现逻辑**:
1. 缓存当前 body 的操作码指针，避免每轮迭代的 `RefCell` 开销
2. 主循环：
   a. 获取当前线程的 IP (body, index)
   b. 如果 body 切换了，重新获取操作码切片指针（`bodies[index]`）
   c. 如果 IP 超出操作码长度：
      - 如果栈非空，弹出并返回栈顶值（表达式风格脚本）
      - 否则返回 NIL
   d. `unsafe` 读取当前 IP 的操作码
   e. 如果是操作码：调用 `self.opcode(opcode, args)`，检查错误队列和 trap
   f. 如果是普通值：直接压栈，IP 前进
   g. Trap 处理：
      - `Pause` → 返回 NIL（让出控制权）
      - `Return(value)` → 返回值
      - `Bail(value)` → 展开调用栈到根帧，返回值

**性能优化**:
- 缓存操作码切片指针（`cached_body_index` + `opcodes_ptr`）避免每次 `RefCell::borrow`
- `is_empty()` 比 `len() > 0` 更快，用于错误检查
- 使用 `trap.on.get()` + `take()` 避免在不必要时写入

### `run_root(&mut self, body_id: u16) -> ScriptValue` (行 595-625)

从根 body 开始执行脚本。

**实现逻辑**:
1. 从 `bodies[body_id]` 提取 scope 和 me 对象
2. 创建根 `CallFrame`（空基址、无返回 IP）
3. 推入作用域和 `me` 上下文
4. 设置 IP 为 body 的开头（index=0）
5. 调用 `run_core()` 进入解释循环

### `call_apply_transform(&mut self, value: ScriptValue) -> Option<ScriptValue>` (行 629-673)

检查值是否有 apply 变换并调用。

**实现逻辑**:
1. 如果是 object，检查对象的 tag 是否有 `as_apply_transform()` 返回的原生函数 ID
2. 如果是 array，检查 array 的 tag
3. 获取函数指针，暂停线程，调用变换函数，恢复线程
4. 返回变换后的值

Apply 变换用于 Lazyd 值、计算属性等需要延迟求值的场景。

### `resume(&mut self) -> ScriptValue` (行 675-678)

恢复暂停的线程（取消 `is_paused` 标记），继续执行 `run_core()`。

用于协程风格的逐步执行。

### `cast_to_f64(&self, v: ScriptValue) -> f64` (行 680-684)

将 ScriptValue 转换为 f64，委托给 `heap.cast_to_f64`。

### 句柄类型管理 (行 686-759)

| 方法 | 逻辑 |
|------|------|
| `handle_type(id)` | 通过 `native.handle_type` 映射查找 LiveId 对应的句柄类型 |
| `new_handle_type(id)` | 注册新的句柄类型，分配唯一的标签并返回类型 |
| `downcast_handle_gc(handle)` | 将 `ScriptHandle` 向下转换为具体的 Rust 类型引用（通过 `heap.handle_ref`） |
| `add_handle_method(ht, method, args, f)` | 为句柄类型添加方法：`native.add_type_method(ht.to_redux(), ...)` |
| `set_handle_setter(ht, f)` | 为句柄类型设置 setter：`native.set_type_setter(ht.to_redux(), f)` |
| `set_handle_getter(ht, f)` | 为句柄类型设置 getter：`native.set_type_getter(ht.to_redux(), f)` |
| `set_handle_call(ht, f)` | 为句柄类型设置 catch-all 方法调度器 `native.set_type_call(ht.to_redux(), f)` |

`set_handle_call` 是高级功能：当句柄上调用未注册的方法名时，触发此函数。函数签名接收 `(vm, args_object, method_id)`。

### 模块管理 (行 761-767)

| 方法 | 逻辑 |
|------|------|
| `new_module(id)` | 创建新的模块对象：委托到 `heap.new_module(id)` |
| `module(id)` | 查找已有模块：委托到 `heap.module(id)` |

### 堆对象遍历 (行 769-822)

| 方法 | 逻辑 |
|------|------|
| `map_mut_with(object, f)` | 临时取出 object 的 map，调用闭包 `f` 修改后放回 |
| `proto_map_iter_mut_with(object, f)` | 从原型链的最祖先到叶节点，依次访问每个对象 map。先递归处理原型，再处理当前对象 |
| `vec_with(object, f)` | 临时取出 object 的 vec 数组，以不可变引用传递给闭包 |
| `vec_mut_with(object, f)` | 临时取出 object 的 vec 数组，以可变引用传递给闭包 |

这些方法通过 swap 操作避免借用冲突，允许在闭包中同时使用 `self.bx.heap` 和其他字段。

### 字符串操作 (行 824-850)

| 方法 | 逻辑 |
|------|------|
| `string_with(value, f)` | 如果 value 是堆字符串，克隆字符串内容以 `&str` 传入闭包；如果是内联字符串，直接传入；否则返回 None |
| `new_string_with(f)` | 创建新字符串：尝试从 `strings_reuse` 池中获取已释放的字符串，调用闭包填充内容，然后插入堆 |

`new_string_with` 的 `strings_reuse` 机制复用已释放的字符串缓冲区，减少堆分配。

### 原生方法注册 (行 852-866)

`add_method(module, method, args, f)`: 在指定模块上注册原生方法。

**实现逻辑**: 委托到 `native.add_method(heap, module, method, args, f)`。

### 全局变量注入 (行 868-901)

| 方法 | 逻辑 |
|------|------|
| `apply_injected_globals_to_scope(scope_obj)` | 将 `injected_globals` 中的所有全局变量写入指定作用域 |
| `apply_injected_globals_to_all_scopes()` | 遍历所有 body 的作用域，逐个注入全局变量 |
| `set_injected_global(key, value)` | 注册或更新全局变量。更新后自动将所有已有作用域更新为该值 |

`set_injected_global` 用于热重载场景：当宿主需要注入新的全局值（如 UI 主题变量）时，所有已编译的脚本立即可见。

### `add_apply_transform_fn(f)` (行 905-910)

注册一个 apply 变换函数。返回 `NativeId` 供后续赋值给对象的 tag。

### `add_script_mod(&mut self, new_mod: ScriptMod) -> u16` (行 912-980)

注册并缓存一个脚本模块。

**实现逻辑**:
1. 提取 crate 名称并注册到 `crate_manifests` 映射
2. 创建新的作用域对象（`scope` 原型），包含 `mod` 引用和注入的全局变量
3. 创建 `me` 对象
4. 检查是否有 `script_mod_overrides` 覆盖的代码
5. 查找是否已存在匹配的 body（相同 file/line/column）：
   - 如果找到：更新 source、scope、me，仅在代码或常数值变化时重新初始化 tokenizer/parser
   - 如果未找到：创建新的 ScriptBody 并追加到 bodies 列表
6. 返回 body 在列表中的索引

**热重载支持**: 匹配已有 body 时，保留 tokenizer/parser 状态（减少重新编译成本），仅在实际代码变化时重置。

### `eval(&mut self, script_mod: ScriptMod) -> ScriptValue` (行 982-984)

最简求值入口，`source = ScriptObject::ZERO`。

### `eval_with_source(&mut self, script_mod, source) -> ScriptValue` (行 986-1037)

带 source 对象的求值入口。

**实现逻辑**:
1. 调用 `add_script_mod` 注册模块并获得 body_id
2. 如果 source 非零：
   - 如果 source 有 `FROM_EVAL` 标记，使用其原型
   - 在 scope 上设置 `__script_source__` 变量
3. 获取 body，第一次执行时：
   - 初始化 tokenizer 和 parser
   - 调用 `tokenizer.tokenize()` 进行词法分析
   - 调用 `parser.parse()` 进行语法分析并生成操作码
4. 调用 `run_root(body_id)` 执行
5. 如果返回值是对象，标记 `FROM_EVAL` 标志
6. 返回结果

**注意**: tokenization 和 parsing 仅第一次执行，之后 body 缓存操作码（除非热重载）。

### `eval_with_append_source(&mut self, script_mod, code, source) -> ScriptValue` (行 1046-1152)

**增量求值**（流式 REPL）。支持逐步输入代码片段（如 Makepad Studio 的交互式脚本编辑）。

**实现逻辑**:
1. 查找已有 body（匹配 file/line/column）
2. 如果未找到，调用 `add_script_mod` 创建
3. 设置 `__script_source__` 变量
4. 恢复解析器检查点（移除上次的自动闭合操作码）
5. 检测内容变化：
   - `content_changed`：如果新代码前缀与已有 tokenizer 内容不符，完全重置并重新 tokenize
   - 仅追加：只 tokenize 新增的字符（`code[prev_len..]`）
6. 处理未完成字符串：`tokenizer.intern_unfinished_string()` 将部分输入的字符串作为值插入操作码，支持实时渲染
7. 调用 `parser.parse_streaming()` 继续解析
8. **关键**: `silence_errors = true` — 不完整的代码必然产生错误，在流式输入期间静默
9. 执行 `run_root(body_id)`，恢复 `silence_errors`，返回结果

流式求值用于 Makepad Studio 的实时编辑面板：用户在输入时，VM 持续解析并执行已完整的部分，呈现即时反馈。

---

## 测试

### `script_apply_eval_refreshes_interpolated_values_on_reused_callsite` (行 1225-1249)

验证 `script_apply_eval!` 宏在重复调用时能正确更新插值变量的值。

**逻辑**: 循环 6 次，每次使用不同的 `is_even_f` 值（1.0 或 0.0），调用 `script_apply_eval!` 更新 item 的 `is_even` 字段，断言值正确。这个测试检查宏是否在每次调用时都刷新了插值变量，而不是缓存第一次的结果。

---

## 关键设计决策

1. **单线程解释器**: VM 运行在单线程上下文中，`ScriptThreads` 管理协程式切换（pause/resume），而非并行执行。
2. **RefCell 内部可变性**: `ScriptCode.bodies` 和 `ScriptCode.native` 使用 `RefCell` 包裹，允许在不持有 `&mut self` 的情况下修改。`run_core` 通过指针缓存绕过 `RefCell` 开销。
3. **指针缓存优化**: `run_core` 缓存 `opcodes_ptr` 和 `opcodes_len`，避免每轮迭代的 `RefCell::borrow`。
4. **增量求值**: `eval_with_append_source` 支持流式输入，专为 Makepad Studio 的交互式编辑设计。
5. **热重载**: `add_script_mod` 的 body 匹配机制和 `script_mod_overrides` 支持无缝热重载。
6. **错误分层**: 运行错误通过 `trap.on`（`Pause`/`Return`/`Bail`）和错误队列两条路径处理，try/catch 在字节码层面实现。

---

## 与 value.rs 的关系

- `ScriptVm` 大量使用 `ScriptValue` 作为值和栈元素
- `ScriptIp` 用于追踪字节码位置（在 value.rs 中定义）
- `run_core` 执行的操作码是 `ScriptValue::OPCODE` 类型
- 错误值通过 `script_err_*!` 宏构造（基于 `ScriptValueType::ERR_*`）

---

## 文件信息
- **路径**: `platform/script/src/vm.rs`
- **行数**: 1250
- **核心类型**: `ScriptVm<'a>`, `ScriptVmBase`, `ScriptCode`, `ScriptBody`
