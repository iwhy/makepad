# `reload_defaults.rs` — 重载默认值集成测试

## 文件位置
`platform/script/tests/reload_defaults.rs`

---

## 总体职责
这个文件测试 Splash 脚本类型系统在热重载（`Apply::Reload`）场景下的默认值行为。它验证了三个关键场景：当脚本端缺少某个字段定义时，Rust 侧的现有值应当保留；但当字段的类型定义了 `set_type_default` 后，缺失字段应回退到类型默认值；以及嵌套对象在重载时也能正确地从类型默认恢复值。

---

## 测试用数据结构

### `ReloadEnumTest`
- `#[derive(Script, ScriptHook)]` 枚举，两个变体：`Fill`（标记为 `#[pick]`，即默认变体）和 `Fixed`。
- 用于测试枚举类型的重载默认值行为。

### `ReloadEnumHolderTest`
- `#[derive(Script, ScriptHook)]` 结构体，包含一个 `#[live] value: ReloadEnumTest` 字段。
- 用于验证枚举字段在重载时的行为。

### `ReloadObjectInnerTest`
- `#[live(1.0)] value: f64` 和 `#[rust(7u32)] rust_value: u32`。
- `#[rust]` 字段应当不受脚本重载影响。

### `ReloadObjectOuterTest`
- 包含 `#[live] inner: ReloadObjectInnerTest`，用于测试嵌套对象重载。

---

## 测试辅助函数

### `fn test_vm() -> ScriptVm<'static>`
- 创建 `ScriptVm`，host 和 std 都是可泄漏的 `Box::leak(Box::new(0i32))` 静态引用。
- 适用于不需要标准库功能的简洁测试环境。

---

## 测试用例

### `reload_missing_live_field_without_type_default_keeps_existing_value()`
- 注册 `ReloadEnumHolderTest::script_api(vm)`，但不设置类型默认值。
- 创建一个 Rust 侧 `holder`，其 `value` 字段为 `ReloadEnumTest::Fixed`。
- 创建一个空的脚本端 holder 原型值（`new_with_proto(holder_api)`），然后在 `Apply::Reload` 模式下应用。
- **断言**：`holder.value` 仍然等于 `ReloadEnumTest::Fixed`。因为未设置类型默认值，脚本端缺失的字段定义不会覆盖 Rust 侧已有的值。

### `reload_missing_enum_field_with_type_default_uses_pick_variant()`
- 注册 `ReloadEnumTest::script_api(vm)` 并设置类型默认值（`set_type_default`）。
- 同样创建 `holder`，其 `value` 为 `Fixed`。
- 注册 `ReloadEnumHolderTest`，创建空原型值，应用 Reload。
- **断言**：`holder.value` 变为 `ReloadEnumTest::Fill`（`#[pick]` 变体）。因为设置了类型默认值，脚本端缺失的字段会被类型默认值填充。

### `reload_missing_object_field_with_type_default_refreshes_from_type_default()`
- 注册 `ReloadObjectInnerTest::script_api(vm)`，创建默认对象，将其 `value` 字段设置为 `42.0`，然后 `set_type_default`。
- 创建 `outer`，其 `inner.value` 为 `1.0`（手动设置）且 `rust_value` 为 `99`。
- 注册 `ReloadObjectOuterTest::script_api(vm)`，创建空原型值，应用 Reload。
- **断言**：
  - `outer.inner.value` 变为 `42.0`（从类型默认恢复）。
  - `outer.inner.rust_value` 仍为 `99`（`#[rust]` 字段不受影响）。

**关键结论：** 这三个测试用例共同验证了 Splash 热重载系统的核心设计原则——"保留现有值，直到脚本明确提供新值，除非类型默认被显式设置"。
