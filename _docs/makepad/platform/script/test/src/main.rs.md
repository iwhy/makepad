# `main.rs` — Splash 脚本语言综合测试平台

## 文件位置
`platform/script/test/src/main.rs`

---

## 总体职责
这个文件是 `makepad_script_test` 二进制 crate 的入口，也是 Splash 脚本语言（`makepad_script` crate）最全面、最复杂的测试套件。它在单个 `main()` 函数中通过 `ScriptVm` 直接执行数十个 Splash 脚本代码块，覆盖了语法解析、类型系统、运算符、控制流、闭包捕获、协程、着色器编译器、正则表达式、HTML 查询、GC 垃圾回收、脚本属性应用（`script_apply_eval!`）和增量（streaming）解析器。

文件总长 4514 行，其中约 4400 行是内联在 Rust 字符串中的 Splash 测试脚本。

---

## Rust 侧测试基础设施

### 测试用数据结构
文件在 Rust 侧定义了一系列用于测试的类型：

- **`StructTest`** —— 测试基本类型支持：`f64` 字段、枚举（`EnumTest`）、`Option<f64>`、`Vec<u8>`。实现了 `on_proto_methods` 钩子，注册了：
  - `return_two()` —— 返回 `2.into()`，测试简单对象方法。
  - `return_three()` —— 句柄方法，测试句柄类型的方法调用。
  - `return_handle()` —— 返回一个 `DummyHandle` 句柄，测试句柄创建和类型注册。
  - `return_headers()` —— 返回 `BTreeMap<String, Vec<String>>`（类似 HTTP headers），测试从 Rust 到 Splash 的复杂类型转换。
- **`EnumTest`** —— 测试枚举的三种变体：`Bare`（单元）、`Tuple(f64)`、`Named { named_field: f64 }`。
- **`ShaderEnum`** —— `#[repr(u32)]` 枚举，测试 const fn 初始化 `make_val(3)` = 30。
- **`MenuTest`** —— 包含 `Vec<LiveId>` 字段的枚举，复现 MenuItem 的序列化问题。
- **`ShaderTest` / `ShaderTest2`** —— `#[repr(C)]` 结构体，测试带 `#[deref]` 继承的着色器类型。
- **`GpuShaderStageTest`** —— 轻量着色器 stage 测试结构体。
- **`RustUniformBufferTest`** —— 测试 `script_pod` 创建的 Rust 端 POD 类型（`Vec2f`/`Vec2f`/`f32` POD 结构体）。
- **`DrawBgTest`** —— 模拟 `draw_bg` 模式（`script_apply_eval!` 目标），包含 `source: ScriptObjectRef`、`is_even: f32`、`color: Vec4f`。

### VM 初始化
- `main()` 创建一个简单的 `ScriptVm`：`host: &mut 0`、`std: &mut 0`、`bx: Box::new(ScriptVmBase::new())`。
- 所有测试脚本通过 `vm.eval(code)` 执行，`code` 是 `script!` 宏构建的 `ScriptMod`。

---

## 测试覆盖范围

### 1. 基础语法与运算符测试（~第 184-346 行）

**算术运算：** `+`、`-`、`*`、`/`、`%`、一元 `-`、`!`、`<<`、`>>`、`&`、`|`、`^`、复合赋值（`+=`、`-=`、`*=`、`/=`、`%=`、`&=`、`|=`、`^=`、`<<=`、`>>=`、`?=`）。
**运算符优先级：** 验证 `2 + 3 * 4 == 14`、`(2 + 3) * 4 == 20` 等优先级规则。
**类型检查（`is`）：** `5 is number`、`"hi" is string`、`true is bool`、`nil is nil`、`{x:1} is object`、`#f00 is color`、`[1 2] is array`。

**短路求值测试（~第 229-266 行）：**
- `||` —— 第一个操作数为 true 时不计算第二个（通过副作用计数器验证）。
- `&&` —— 第一个操作数为 false 时不计算第二个。
- `|?`（nil 合并）—— 第一个操作数非 nil 时不计算第二个。

**字段和索引复合赋值：**
- 对象字段：`x.f += 2`、`x.f ?= 5`。
- 数组索引：`x[1] += 2`、`x[1] ?= 5`。

### 2. 控制流测试（~第 347-352 行）

- `for x in 4{...}` —— 数字范围循环。
- `for x in 7{ if x == 3 || x == 5 continue; ... }` —— `continue` 跳过。
- `loop{... break}` —— 无限循环 + break。
- `while c < 9 c += 1` —— while 循环（无大括号）。
- `if x < y && y > 0 { ... }` —— if 条件中的 `&&`/`||` 与大括号解析的歧义测试。

### 3. 对象冻结与原型系统（~第 368-395 行）

- **`freeze_api()`**：冻结后不能设置未知属性，不能修改已知属性值；但可以通过继承（`{x{x: 3}}`）修改。
- **`freeze_module()`**：不能修改现有属性，不能添加新属性，不能通过继承添加 vec 项。
- **`freeze_component()`**：完全禁止写操作，但允许通过派生创建同类型值（类型检查）。

### 4. 作用域与闭包捕获测试（~第 397-402 行）

- `let x = 1; let f = || x; let x = 2; let g = || x; assert(f() == 1); assert(g() == 2);` —— 验证闭包捕获的是定义时的值而非引用。

### 5. Try/Ok 错误处理（~第 405-406 行）

- `try{undef = 1} assert(true) ok assert(false)` —— 访问未定义变量抛出异常。
- `let t = 0 try{t = 1} assert(false) ok assert(true)` —— 正常执行进入 `ok` 分支。

### 6. Rust-Splash 类型系统测试（~第 408-466 行）

- **StructTest**：`try{s{field:5}}` 合法，`try{s{field:"HI"}}` 类型错误。
- **枚举类型检查**：`Tuple(f64)` 只接受单个 f64 参数，`Named{named_field: f64}` 只接受数字命名字段。
- **Option 类型**：`s{opt:nil}` / `s{opt:1.0}` 合法，`s{opt:"false"}` 类型错误。
- **Vec 类型**：`s{vec:[1 2 3 4]}` 合法，`s{vec:[false]}` 类型错误。

### 7. 字符串操作测试（~第 468-473 行）

- `"hi".to_bytes().to_string() == "hi"` —— Uint8Array 与字符串转换。
- `"hi".to_chars().to_string() == "hi"` —— 字符数组与字符串转换。

### 8. JSON 序列化与解析（~第 475-500 行）

- 对象 → `to_json()` → `parse_json()` 往返测试。
- OpenAI SSE chunk 解析：标准 JSON 格式和 `"data: "` 前缀 SSE 格式。

### 9. Do 链与回调测试（~第 502-504 行）

- `f(1) do |x| x+1` —— `do` 语法糖将函数作为最后一个参数传入。

### 10. Nil 安全访问测试（~第 507-514 行）

- `ok{x.y.z}` —— 错误抑制操作符，返回 nil 而非 panic。
- `x.c.?d` —— nil 安全字段访问，如果 c 为 nil 返回 nil 而非 panic。

### 11. 字符串连接测试（~第 516-525 行）

- `x.t += "b" + "c" + 2` —— 字符串 + 数字自动转换。
- 数组元素字符串连接。

### 12. 函数定义测试（~第 527-537 行）

- 闭包：`|x| x + 1`。
- `fn` 关键字：`fn x{3}`、`fn x(a = 2){a + 2}`（默认参数）。
- 多参数函数：`fn test(a,b){a+b}`。

### 13. Return-in-if 转义分析测试（~第 539-644 行）

解释器（非着色器）的 return-in-if 测试，覆盖 10 种模式：
1. if-return，无 else，代码随后。
2. if-return，else-return（两个分支都 return）。
3. if-return，else fallthrough。
4. if fallthrough，else-return。
5. if-else if-else 链。
6. if-else if（无最终 else）+ fallthrough。
7. 嵌套 if。
8. 深层嵌套。
9. 只有单个分支有 return 的嵌套。
10. 早期 return vs 表达式结果。

### 14. For 循环析构测试（~第 646-870 行）

**标准风格：** `for v in set` / `for k v in set` / `for i v in arr` / `for i k v in obj`。
**括号风格：** `for (v) in set` / `for (i, v) in arr` / `for (i, k, v) in obj`。
**支持的类型：**
- 数字范围：`0..5`。
- 数组：`[10, 20, 30]`。
- vec-based 对象：`{a := 1, b := 2, c := 3}`。
- map-based 对象：`{"alpha": 1, "beta": 2}`。
- 混合类型。

**for 循环中的 return：** 验证嵌套 for 循环中的早期 return 正确工作。
**`:=` 属性测试（第 1019-1175 行）：**
- `:=` 存储在 vec（有序，可迭代） vs `:` 存储在 map（hash 存储不可迭代）。
- `vec_len()` / `map_len()` / `vec_key(n)` 内省 API。
- `+: ` 合并操作符在 `:=` 属性上的行为。
- 原型链中的 `:=` 属性读取和写入。

### 15. 析构赋值测试（~第 1196-1272 行）

- 懒惰 `?=`：有值时不执行 RHS，`nil` 时执行。
- 数组析构：`let [a, b] = [1, 2]`。
- 对象析构：`let {x, y} = {x: 3, y: 4}`。
- 默认值（懒惰求值）：`let {a, b=expensive_fn()} = {a: 1}`。
- 嵌套模式：`[{x}]`、`[[a, b]]`、`[s, {t}]`。

### 16. For 循环闭包捕获测试（~第 1278-1305 行）

- 验证每个迭代创建一个独立捕获副本而非共享变量。
- `for i in 0..3 { fns.push(|| i) }; assert(fns[0]() == 0); assert(fns[1]() == 1)`。

### 17. 着色器编译器综合测试（~第 1307-2045 行）

这是最庞大的测试块，定义了一个全面的 `ShaderTest2` 着色器并调用 `shader.test_compile_draw()` 编译。

**Shader 声明测试：**
- 所有着色器管道元素：`vertex_pos`、`pixel`（fragment output）。
- 顶点缓冲区（`vtx`）、实例数据（`inst_pos`/`inst_scale`/`inst_color`/`inst_id`）、uniforms（`u_time`/`u_scale`/`u_color`/`u_enabled`/`u_count`/`u_flags`）。
- Uniform 缓冲区（`uniforms`）、纹理（`tex_diffuse`/`tex_normal`）、varyings（`v_uv`/`v_color`/`v_intensity`/`v_normal`/`v_world_pos`）。
- Helper 函数（闭包风格和 fn 风格）。

**Vertex 着色器测试（第 1591-1618 行）：**
- 顶点缓冲区访问、实例数据读取、uniform 缓冲区访问。
- 设置 varyings。
- Void return-in-if（最简单情况）。

**Fragment 着色器测试（第 1620-2040 行）：**
- 全类型算术运算：`f32`、`i32`、`u32`、`f16`（半精度）。
- 向量类型算术：`vec2`/`vec3`/`vec4` 的 `+`、`-`、`*`、`/`、一元 `-`。
- 标量与向量混合运算。
- 比较运算和逻辑运算。
- 复合赋值：所有类型上的 `+=`/`-=`/`*=`/`/=`/`%=`/`&=`/`|=`/`^=`/`<<=`/`>>=`。
- 结构体字段赋值和复合赋值。
- 数组索引（字面量和变量索引）。
- `if/else` 表达式和语句。
- `match` 表达式（枚举匹配 + 通配符）。
- 循环：`for`、`loop`、`while`、嵌套循环、`break`/`continue`。
- 内置函数：`abs`/`floor`/`ceil`/`round`/`fract`/`sqrt`/`sin`/`cos`/`tan`/`asin`/`acos`/`atan`/`exp`/`log`/`exp2`/`log2`/`min`/`max`/`pow`/`step`/`atan2`/`clamp`/`mix`/`smoothstep`/`length`/`normalize`/`dot`/`cross`/`distance`（标量和向量版本）。
- 函数调用（自我调用 helper 函数）。
- 所有声明类型：`let`/`var`、`f32`/`f16`/`i32`/`u32`/`bool`/`vec2`/`vec3`/`vec4`/`vec2i`/`vec3i`/`vec4i`/`vec2u`/`vec3u`/`vec4u`/颜色。
- Swizzle：`.x`/`.xy`/`.xyz`/`.xyzw`/`.wzyx`/`.xxxx`/`.rg`/`.rgb`。
- 颜色字面量：`#ff0000` / `#00ff00` / `#0000ff` / `test_color`。
- Varying/uniform/纹理采样访问。
- 位运算：`asuint`/`asint`/`asfloat`/移位。
- Scope uniforms 访问。
- 光照计算（点积法线）。
- `TestSdf` struct 方法测试（构造函数、self 突变、交叉调用）。
- Return-in-if 转义分析（18 种模式，包括 void return、for/while/loop 中的 return）。

**GPU 多阶段实验着色器（第 2049-3768 行）：**
包含 20+ 个独立 GPU 着色器编译测试（`gpu_stage_0` 到 `gpu_stage_4j`），逐步增加复杂度：
- `gpu_stage_0`：基础渐变着色器。
- `gpu_stage_1`：双精度模拟（double-single）算法（ds_add/ds_mul/ds_div/ds_box_fold/ds_sqrt 等，约 130 行）。
- `gpu_stage_1a`：标量 helper 通过函数调用的类型推断。
- `gpu_stage_1b`：嵌套标量 helper 链（ds_quick_two_sum/ds_split/ds_two_prod）。
- `gpu_stage_2`：混合分形 DE 循环（16 次迭代，两个分形 slot，for 循环）。
- `gpu_stage_2a-2e`：循环基础的 return-in-if 模式（无返回、直接返回、if-else 返回、calc_de 内联）。
- `gpu_stage_3`：完整 ray_march 循环（位置计算、步进逻辑、边界检查）。
- `gpu_stage_4`：ray_march 细化循环（refinement bisection）。
- `gpu_stage_4a-4j`：边缘情况——vec2 if 表达式、Rust POD uniform buffer、var 重赋值、分支局部 var、外部 var 被分支局部 helper 捕获、helper 调用上的单元否定、bool-and + else-return。

**特殊测试：** `shader.test_compile_draw_contains` 检查编译输出中包含 `"<< uint(3)"` 和 `">> uint(31)"`。

### 18. 正则表达式测试（~第 3771-4106 行）

**构造函数：** `regex("pattern", "flags")` 创建、类型检查（`is_regex()`）、`source` 属性、内联缓存（相同模式+标志返回相同对象）。

**方法测试：**
- `re.test(str)` —— 匹配测试。
- `re.exec(str)` —— 执行并返回匹配对象（`value`/`index`/`captures`）。
- `str.search(re)` / `str.search("literal")` —— 返回索引或 -1。
- `str.match_str(re)` —— 非全局返回详情对象，全局返回数组。
- `str.match_all(re)` —— 返回所有匹配的详情对象数组。
- `str.split(re)` —— 按正则分割，支持捕获组（捕获包含在结果中）。
- `str.replace(re, replacement)` —— 支持 `$&`（全匹配）、`$1`/`$2`（捕获组）、`$$`（字面 `$`）、全局/非全局模式。

**边缘情况：**
- 空模式、交替、量词（`a{2,4}`）、字符类（`[a-z]+`）、嵌套组、无匹配时保持原字符串、GC 交互（100+ 次创建后仍可用）。

### 19. HTML 解析与查询测试（~第 4108-4173 行）

使用 `str.parse_html()` 解析 HTML，然后用 `doc.query(selector)` 进行 CSS 类查询。

**支持的查询语法：**
- `doc.query("p")` —— 标签选择器（返回元素集）。
- `doc.query("p[0]")` —— 索引选择器。
- `doc.query("span").text` —— 文本内容。
- `doc.query("div@class")` —— 属性值。
- `doc.query("p.bold")` —— 类名选择器。
- `doc.query("p.text")` —— 文本数组。
- `doc.query("p@class")` —— 属性值数组。
- `doc.query("p").array()` —— 转为数组。
- `doc.query("div").query("p")` —— 链式查询。
- `doc.query("div > p")` —— 直接子元素。
- `doc.query("ul a")` —— 后代元素。
- `doc.query("div > *")` —— 所有子元素（通配符）。
- `doc.query("#target")` —— ID 选择器。
- `doc.query("p#target")` —— 标签+ID 组合。
- `doc.attr("class")` —— 文档根属性。

### 20. GC 垃圾回收压力测试（~第 4175-4335 行）

- **简单循环创建：** 1000 次迭代创建弃用对象。
- **嵌套对象：** 500 次三层嵌套。
- **函数创建：** 100 次调用创建 50 元素的数组。
- **递归创建：** 构建深度为 6 的二叉树 20 次。
- **字符串操作：** 多次 to_bytes() 转换。
- **数组操作：** push/pop 200 次。
- **JSON 往返：** 创建记录 → to_json() → parse_json() 300 次。
- **闭包捕获：** 200 次创建捕获闭包。
- **原型链：** 500 次基于原型创建派生对象。
- **存活对象验证：** GC 后关键对象仍然有效。
- **静态容器（`gc.set_static`）：** 不可变验证——不能写入、添加、删除、修改。
- **字符串键存活：** JSON 解析后的字符串键在 GC 后正确保留。

### 21. script_apply_eval! 压力测试（~第 4336-4370 行）

- 创建 10 个 `DrawBgTest` 实例，每个有 `ScriptObjectRef`。
- 模拟 PortalList 的 draw 循环模式：1000 次迭代，每次对每个 item 调用 `script_apply_eval!` 设置 `is_even` 属性。
- 每 100 次迭代运行一次 GC，验证 GC 与 script_apply_eval 的交互没有内存损坏。

### 22. 性能基准测试（~第 4371-4380 行）

- 计算斐波那契数 `fib(20)`（递归实现），测量执行时间并打印 "Duration {secs}"。

### 23. 增量（Streaming）解析器测试（~第 4382-4513 行）

**Part 1 — Opcode 比较（~第 4414-4465 行）：**
- 使用 `ScriptTokenizer` + `ScriptParser` 逐字节增量解析一个 400+ 字符的 Splash 代码片段。
- 在每个字节后，使用 `restore_checkpoint` + `parse_streaming` 进行增量解析。
- 与全量（非增量）解析的 token 序列和 opcode 序列进行完全比较。
- 如果有任何不匹配，打印详细的 diff（opcode 索引、ref vs inc 值、差异标记）。

**Part 2 — 增量 eval_with_append_source（~第 4468-4485 行）：**
- 逐字节使用 `vm.eval_with_append_source` 执行同一段代码。
- 只验证不 panic/crash（不完整代码的错误是预期的）。

**Part 3 — 20 字符块模拟 AI 流式输出（~第 4487-4513 行）：**
- 以 20 字符为块（对齐 Unicode 边界）逐步追加和评估代码。
- 验证流式执行路径不会 panic 或产生错误结果。
