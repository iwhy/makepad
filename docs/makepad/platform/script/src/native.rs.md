# `native.rs` 源码解读

**路径:** `platform/script/src/native.rs`
**行数:** 481
**核心职责:** 提供 Rust 原生函数在脚本引擎中的注册机制，包括丰富的辅助宏和 `ScriptNative` 类型系统（getter/setter/method/call 的注册）以及所有基础类型的 `.ty`、`is_*`、`to_json`、`to_number`、`to_string` 等方法注册。

---

## 辅助宏

### `script_value_f64!`（第13-26行）
从 args 对象中提取指定命名参数或索引参数并转为 f64。两种形式：
- `script_value_f64!(ctx, args.id)`：按 id 查找
- `script_value_f64!(ctx, obj[index])`：按索引查找 vec 值

### `script_value_bool!`（第29-42行）
同理，从 args 或 vec 中提取 bool。

### `script_value!`（第45-81行）
通用的值提取宏，四种形式：
- `$vm, $obj.$id`：一层属性查找
- `$vm, $obj.$id.$id2`：两层属性查找
- `$vm, $obj[$index]`：vec 索引查找
- `$vm, $obj as array[$index]`：数组索引查找

### `script_has_proto!`（第84-93行）
检查某对象是否在原型链中包含指定 proto。

### `script_is_fn!`（第96-100行）
检查某对象是否是函数。

### `script_array_index!`（第103-110行）
安全的数组索引访问。

### `set_script_value!`（第113-126行）
设置属性或 vec 索引值。两种形式：`.` 属性和 `[]` 索引。

### `set_script_value_to_api!`（第128-144行）
设置属性值为某个类型的 `script_api` 结果。

### `set_script_value_to_pod!`（第147-168行）
设置属性值为某个类型的 `script_pod` 结果。

### `script_args!`（第171-174行）
创建 `(LiveId, ScriptValue)` 参数列表的语法糖：`script_args!(x=1.0, y=2.0)`。

### `script_args_def!`（第177-181行）
与 script_args 类似，但使用 `id_lut!`（编译期查询表）。

---

## 类型别名

**路径:** 第184-191行

- `NativeGetterFn`：`Box<dyn Fn(&mut ScriptVm, ScriptValue, LiveId) -> ScriptValue>`，用于属性 getter
- `NativeSetterFn`：`Box<dyn Fn(&mut ScriptVm, ScriptValue, LiveId, ScriptValue) -> ScriptValue>`，用于属性 setter
- `NativeCallFn`：`Box<dyn Fn(&mut ScriptVm, ScriptObject, LiveId) -> ScriptValue>`，用于 catch-all 方法调用
- `NativeFn`：`Box<dyn Fn(&mut ScriptVm, ScriptObject) -> ScriptValue>`，用于普通函数

---

## 结构体 `ScriptNative`

**路径:** 第193-201行

原生函数和目标分发表：
- `functions: Vec<NativeFn>`：所有注册的原生函数
- `type_table: Vec<LiveIdMap<LiveId, ScriptObject>>`：按类型 redux 索引的方法表
- `handle_type: LiveIdMap<LiveId, ScriptHandleType>`：句柄类型注册表
- `getters/setters: Vec<NativeGetterFn/SetterFn>`：按类型 redux 索引的 getter/setter
- `calls: Vec<Option<NativeCallFn>>`：按类型 redux 索引的 catch-all call

---

## impl ScriptNative

### `new(h)`（第204-211行）

初始化 `ScriptNative`，依次注册共享方法、Object 类型方法、Array 类型方法和 String 类型方法。

### `add_fn(heap, args, f)`（第215-226行）

泛型函数注册：将闭包装箱后委托给非泛型 `add_fn_boxed` 以减少单态化膨胀。

### `add_fn_boxed(heap, args, f)`（第230-253行）

核心函数注册逻辑：
1. 分配函数索引
2. 创建 proto 为 `id!(native)` 的新对象
3. 设置存储类型为 `vec2`（用于参数）
4. 设置函数指针（`ScriptFnPtr::Native(NativeId{index})`）
5. 将每个默认参数写入函数对象的 map
6. 压入 functions 列表

### `add_apply_transform_fn(f)`（第257-266行）

注册一个 apply_transform 函数（接受 ScriptObject 返回 ScriptValue），返回 NativeId 供引用。

### `add_method(heap, module, method, args, f)`（第268-281行）

在指定 module 对象上注册一个方法：创建函数对象并 `set_value_def`。

### `new_handle_type(heap, id)`（第283-295行）

注册新句柄类型：分配类型索引（使用 `REDUX_HANDLE_FIRST` 起始的范围），并将 `ty` 方法注册到该类型。

### `set_type_getter(ty_redux, f)` / `set_type_setter(ty_redux, f)`（第297-309行）

为指定类型设置属性 getter/setter。

### `set_type_call(ty_redux, f)`（第311-317行）

为指定类型设置 catch-all 调用处理函数。

### `ensure_type_table_capacity(ty_redux)`（第321-347行）

确保 type_table、getters、setters、calls 向量有足够容量。不足时 resize 并用默认错误处理函数填充空隙。

### `add_type_method(heap, ty_redux, method, args, f)`（第349-362行）

在指定类型的类型表上注册方法。

---

## `add_shared`（第364-479行）

为所有基础类型注册共享方法：

### `.ty` 属性（第365-400行）
每个类型注册 `ty` 方法返回该类型的标识 id：
number/nan/bool/nil/color/string/object/array/regex/opcode/err/id

### 通用类型方法（第402-440行）
对每种类型注册三个通用方法：
- **`to_json`**：调用 `heap.to_json` 序列化为 JSON 
- **`to_number`**：通过 `heap.cast_to_f64` 转为数字
- **`to_string`**：非数组类型注册，返回字符串表示

### `is_*` 类型查询（第442-479行）
每种类型注册所有类型的 `is_*` 查询方法（如 `number.is_string()` 返回 false）。这样所有脚本值都可以调用任何类型的查询方法。
