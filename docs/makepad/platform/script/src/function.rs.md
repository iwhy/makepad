# `function.rs` 源码解读

**路径:** `platform/script/src/function.rs`
**行数:** 186
**核心职责:** 定义脚本引擎中的函数指针系统（`ScriptFnPtr`），区分脚本函数和原生函数；提供函数参数管理的核心方法（命名/未命名参数的注入及类型检查）。

---

## 结构体 `NativeId`

**路径:** 第7-9行

原生函数的标识：`index: u32`，指向 `ScriptNative::functions` 列表中的索引。

---

## 结构体 `ScriptFnRef`

**路径:** 第11-24行

脚本函数引用的包装类型，内部持有 `ScriptObjectRef`。实现了 `From<ScriptFnRef> for ScriptValue`（转为 object 类型的 ScriptValue）。

### `as_object(&self)`（第21-23行）
解引用为底层的 `ScriptObject`。

---

## 枚举 `ScriptFnPtr`

**路径:** 第26-30行

函数指针的两种变体：
- `Script(ScriptIp)`：脚本函数，存储指令指针（指向字节码中函数体的起始位置）
- `Native(NativeId)`：原生 Rust 函数，存储 NativeId

---

## `ScriptRefOptionExt` for `Option<ScriptFnRef>`

**路径:** 第32-39行

为 `Option<ScriptFnRef>` 提供扩展：`as_object()` 返回 `Option<ScriptObject>`。

---

## `ScriptHeap` 上的函数操作方法

### `set_fn(ptr, fnptr)`（第45-48行）

在指定对象上设置函数指针：`object.tag.set_fn(fnptr)`。

### `as_fn(ptr)`（第50-53行）

获取指定对象的函数指针（如果它是函数对象）。

### `is_fn(ptr)`（第55-58行）

检查指定对象是否被标记为函数。

### `set_reffed(ptr)`（第60-63行）

将对象标记为"被引用"（防止 GC 回收）。

### `parent_as_fn(ptr)`（第65-73行）

沿原型链向上查找：如果当前对象的 proto 是一个带有函数标签的对象，则返回其函数指针。

---

## 参数管理方法

### `unnamed_fn_arg(top_ptr, value, trap)`（第75-112行）

向函数对象注入未命名参数：
1. 获取函数对象的 map 当前长度作为参数索引
2. 从 proto 对象获取对应位置的默认参数
3. **类型检查**：若默认值非 nil 且与传入值类型 redux 不匹配则报错
4. 将值以 proto 指定的 key 插入 map
5. 若索引超出 proto 的 vec 长度（无对应形参），则视为变长参数追加到 vec

### `named_fn_arg(top_ptr, key, value, trap)`（第114-145行）

向函数对象注入命名参数：
1. 遍历 proto 的 vec 查找匹配 key
2. **类型检查**：若默认值非 nil 且类型 redux 不匹配则报错
3. 若找到则 `map_insert(key, value)`
4. 若未找到则报 `script_err_not_found`

### `push_all_fn_args(top_ptr, args, trap)`（第147-185行）

批量注入所有参数（按位置）：
1. 遍历 args 切片
2. 对每个参数，从 proto vec 中匹配位置和 key
3. **类型检查**：若该位置的默认值非 nil 且类型 redux 不匹配则报错
4. 若位置在 proto vec 范围内则 `map_insert(key, value)`
5. 若超出范围则 push 到 vec（varargs）
