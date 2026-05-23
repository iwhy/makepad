# `platform/src/script/event.rs` — 事件类型脚本注册

## 文件职责

该文件将 Makepad 的 `KeyCode` 枚举注册到 Splash VM 脚本的 `draw` 模块中，使脚本代码可以引用 `draw.KeyCode` 及其变体（如 `KeyCode::A`、`KeyCode::Space`、`KeyCode::Enter` 等）。

---

## `pub fn script_mod(vm: &mut ScriptVm) -> ScriptValue`

```rust
let draw = vm.module(id!(draw));
set_script_value_to_api!(vm, draw.KeyCode);
```

- 通过 `vm.module(id!(draw))` 获取之前 `draw.rs` 注册的 `draw` 模块
- 使用 `set_script_value_to_api!` 宏将 `KeyCode` 枚举注册到 `draw` 模块下

注册之后，脚本中可以这样使用：

```
match key.code {
    draw.KeyCode.A => { ... }
    draw.KeyCode.Space => { ... }
    _ => {}
}
```

`KeyCode` 枚举定义在 `crate::event::KeyCode`，包含所有标准键盘按键的变体。该文件本身不定义任何新类型，仅作为类型导出和注册的桥梁。

返回值 `NIL`。
