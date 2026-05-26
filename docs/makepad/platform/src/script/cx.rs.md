# `platform/src/script/cx.rs` — 平台上下文脚本注册

## 文件职责

该文件将 Makepad 平台层的核心上下文（`Cx`）能力暴露给 Splash VM 脚本。它在脚本中注册了一个 `cx` 模块，使得脚本代码可以调用 `cx.quit()` 退出应用、查询 `cx.os_type` 获取当前操作系统类型。

---

## `pub fn script_mod(vm: &mut ScriptVm)`

创建名为 `cx` 的脚本模块，注册以下内容：

### 类型注册

```rust
set_script_value_to_api!(vm, cx.OsType);
```

将 Rust 枚举 `OsType`（`Windows`、`MacOS`、`Linux`、`Android`、`IOS`、`Web` 等变体）注册到 `cx` 模块下，使脚本可以引用和匹配 `cx.OsType`。

### 方法：`cx.quit()`

```rust
vm.add_method(cx, id_lut!(quit), script_args_def!(), |vm, _args| {
    vm.cx_mut().request_quit(QuitReason::App);
    NIL
});
```

- 通过 `ScriptVmCx` trait 获取 `Cx` 的可变引用
- 调用 `Cx::request_quit(QuitReason::App)` 发起应用退出请求
- 无参数、无返回值

### 方法：`cx.os_type()`

```rust
vm.add_method(cx, id_lut!(os_type), script_args_def!(), |vm, _args| {
    let os_type = vm.cx().os_type().clone();
    os_type.script_to_value(vm)
});
```

- 通过 `ScriptVmCx` trait 获取 `Cx` 的不可变引用
- 克隆当前的 `OsType` 枚举值
- 调用 `script_to_value` 将其转换为脚本可用的值返回
