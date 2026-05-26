# `lib.rs` 源码解读

**路径:** `platform/script/derive/src/lib.rs`
**行数:** 66
**核心职责:** proc-macro crate 的入口文件，声明所有子模块并导出所有公开的 proc-macro 函数

---

## 类型定义

无直接类型定义。文件中仅包含模块声明和宏入口函数。

## impl 块

无 impl 块。所有逻辑委托到各子模块。

## 函数/宏

### `proc_macro fn script(input: TokenStream) -> TokenStream`
**第 13 行**

`script!` 宏的入口。接收输入 TokenStream 后立即委托给 `script::script_impl()` 执行。该宏用于在 Rust 代码中嵌入 Makepad 脚本表达式(如同在运行时执行的 DSL 代码块)，将脚本源码字符串化并包装为 `ScriptMod` 结构体。

### `proc_macro fn script_mod(input: TokenStream) -> TokenStream`
**第 18 行**

`script_mod!` 宏的入口。将输入直接传递给 `script::script_mod_impl()`。该宏扩展为 `pub fn script_mod(vm: &mut ScriptVm) -> ScriptValue` 函数，用于定义完整的脚本模块——包括 UI 组件定义、主题变量、样式声明等。是构建 Makepad UI 界面的核心入口。

### `proc_macro fn script_apply_eval(input: TokenStream) -> TokenStream`
**第 23 行**

`script_apply_eval!` 宏的入口。委托给 `script::script_apply_eval_impl()`。该宏在运行时动态更新 widget 属性的场景中使用，接收 `(cx, target, { script_code })` 参数，编译脚本并在虚拟机中求值后应用到目标 widget 上。

### `proc_macro fn script_err_gen(input: TokenStream) -> TokenStream`
**第 28 行**

`script_err_gen!` 宏的入口。委托给 `error::script_err_gen_impl()`。用于生成脚本运行时错误宏，创建带标准错误信息格式的 `script_err_xxx!` 宏，推送错误信息到 `ScriptTrap`。

### `proc_macro_derive(Script, attributes(...)) fn derive_script(input: TokenStream) -> TokenStream`
**第 49 行**

`#[derive(Script)]` 派生宏的入口。支持的属性包括: `apply_default`、`source`、`new`、`live`、`rust`、`pick`、`splat`、`walk`、`layout`、`deref`、`uid`。委托给 `derive_scriptable::derive_script_impl()`。这是 Makepad 脚本系统中最重要的派生宏，为数据结构自动生成 `ScriptNew`、`ScriptApply`、`ScriptHookDeref`、`ScriptDeriveMarker` 等 trait 实现，使 Rust 类型能够在脚本虚拟机中创建、序列化、反序列化和应用。

### `proc_macro_derive(ScriptHook, attributes()) fn derive_script_hook(input: TokenStream) -> TokenStream`
**第 54 行**

`#[derive(ScriptHook)]` 派生宏的入口。委托给 `derive_scriptable::derive_script_hook_impl()`。为空实现的 `ScriptHook` trait 生成代码，为 struct 或 enum 提供默认的 `on_before_apply`、`on_after_apply`、`on_custom_apply` 等钩子方法实现。

### `proc_macro fn pod_swizzle_vec_match(input: TokenStream) -> TokenStream`
**第 59 行**

`pod_swizzle_vec_match!` 宏的入口。委托给 `swizzle::pod_swizzle_vec_match_impl()`。生成向量编组访问的 `match` 代码，处理 `.x`、`.y`、`.z`、`.w` 及 `.r`、`.g`、`.b`、`.a` 命名空间下所有 1-4 维组合编组操作，将字段名映射到对应的 `pod_swizzle_vec` 调用。

### `proc_macro fn pod_swizzle_vec_type(input: TokenStream) -> TokenStream`
**第 64 行**

`pod_swizzle_vec_type!` 宏的入口。委托给 `swizzle::pod_swizzle_vec_type_impl()`。生成向量编组类型查询的 `match` 代码，对于给定的编组字段名返回其对应的 `SwizzleType`（1-4 维向量类型），用于在脚本类型系统中查找编组操作的返回类型。
