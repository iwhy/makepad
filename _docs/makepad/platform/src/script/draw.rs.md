# `platform/src/script/draw.rs` — 绘制与窗口类型脚本注册

## 文件职责

该文件将 Makepad 的绘制系统类型和窗口配置类型注册到 Splash VM 脚本中。脚本代码可以引用 `draw.MouseCursor`、`draw.WindowBackdrop`、`draw.ScriptDrawPass`、`draw.ScriptWindowHandle` 等类型，用于构建 UI 渲染通道和窗口配置。

---

## `pub fn script_mod(vm: &mut ScriptVm) -> ScriptValue`

创建名为 `draw` 的脚本模块，按以下顺序注册：

### 几何句柄类型

```rust
vm.new_handle_type(id!(geometry));
```

注册一个名为 `geometry` 的句柄类型，供脚本传递几何体的引用。

### 统一数据结构注册（POD）

```rust
set_script_value_to_pod!(vm, draw.DrawCallUniforms);
set_script_value_to_pod!(vm, draw.DrawListUniforms);
set_script_value_to_pod!(vm, draw.DrawPassUniforms);
```

三个宏调用将 `DrawCallUniforms`、`DrawListUniforms`、`DrawPassUniforms` 注册为纯数据（POD）类型，脚本可以构造和读取这些 uniform 数据块。

### 窗口/光标枚举注册（API）

```rust
set_script_value_to_api!(vm, draw.MouseCursor);
set_script_value_to_api!(vm, draw.WindowBackdrop);
set_script_value_to_api!(vm, draw.MacosWindowKind);
set_script_value_to_api!(vm, draw.MacosWindowChrome);
set_script_value_to_api!(vm, draw.MacosWindowLevel);
set_script_value_to_api!(vm, draw.MacosWindowConfig);
```

将六个枚举/结构类型注册为脚本 API 类型：
- `MouseCursor`：光标形状（`Arrow`、`Hand`、`Text` 等）
- `WindowBackdrop`：窗口背景材质
- `MacosWindowKind`：macOS 窗口种类
- `MacosWindowChrome`：macOS 窗口 chrome 样式
- `MacosWindowLevel`：macOS 窗口层级
- `MacosWindowConfig`：macOS 窗口配置结构体

### `ScriptDrawPass` 类型默认值

```rust
let pass_default = ScriptDrawPass::script_api(vm);
vm.bx.heap.set_type_default(pass_default.as_object().unwrap());
set_script_value!(vm, draw.ScriptDrawPass = pass_default);
```

- 调用 `ScriptDrawPass::script_api(vm)` 创建该类型的脚本 API 表示
- 通过 `set_type_default` 将其设置为 VM 堆中的类型默认值
- 存入 `draw.ScriptDrawPass` 供脚本使用

### `ScriptWindowHandle` 类型默认值

```rust
let window_default = ScriptWindowHandle::script_api(vm);
vm.bx.heap.set_type_default(window_default.as_object().unwrap());
set_script_value!(vm, draw.ScriptWindowHandle = window_default);
```

- 类似地注册 `ScriptWindowHandle`，这是脚本中操作窗口的核心句柄类型
- 包含 `window.inner_size`、`window.title` 等属性

### 返回值

```rust
NIL
```

模块注册函数返回 `NIL`。
