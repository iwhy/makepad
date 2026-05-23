# `fs.rs` — 文件系统标准库（只读/写）

## 文件位置
`platform/script/std/src/fs.rs`

---

## 总体职责
这个文件向 Splash 脚本环境注册了 `fs` 模块，提供基本的文件读写能力。所有操作都是同步阻塞的，使用 Rust 标准库的 `std::fs::File` 实现。该模块注册到脚本中的 `mod.fs`。

---

## 关键函数

### `pub fn script_mod(vm: &mut ScriptVm)`
- 在 `vm` 上创建一个名为 `id!(fs)` 的新模块，并向其中添加四个方法。
- 使用 `script_args_def!` 宏定义参数签名，通过 `script_value!` 宏从脚本参数中提取路径和数据。

### `fs.read(path)` / `fs.read_to_string(path)`
- 通过 `for sym in [id_lut!(read), id_lut!(read_to_string)]` 批量注册两个同名方法，它们的实现逻辑完全相同。
- 调用 `vm.string_with(path, |_vm, s| fs::File::open(s).ok())` 尝试以只读方式打开文件。如果打开失败，返回 `None`，则通过 `script_err_io!` 抛出 "file system error"。
- 如果文件成功打开，创建一个新的脚本字符串并将文件内容读取到该字符串中。`read_to_string` 与 `read` 在当前实现中功能等价，都读取完整文件到一个字符串。
- 读取过程中 `read_to_string` 失败（如 UTF-8 错误），同样通过 `script_err_io!` 抛出错误。注意这里使用的 trap 是 `vm.bx.threads.trap()`（异步线程的 trap），而非 `vm.trap()`。

### `fs.write(path, data)` / `fs.write_string(path, data)`
- 同样通过 `for sym in [id_lut!(write), id_lut!(write_string)]` 批量注册，但它们的参数签名有两个：`path` 和 `data`。
- 使用 `fs::File::create(s).ok()` 以写入模式创建/截断文件。如果创建失败抛出 "file system error"。
- 写入数据的类型分两种处理：
  - **字符串类型**（`data.is_string_like()`）：通过 `vm.string_with(data, |vm, s| file.write_all(s.as_bytes()))` 将字符串转为字节写入。
  - **数组类型**（`data.as_array()` 匹配成功）：检查底层存储类型是否为 `ScriptArrayStorage::U8`（Uint8Array），如果是则直接写入原始字节数据；其他存储类型（U16/U32/F32/ScriptValue）抛出类型不匹配错误 "invalid fs arg type"。
- `write` 与 `write_string` 在当前实现中功能等价，都可以写入字符串和 Uint8Array。
- 写入成功返回 `NIL`；任一错误路径都通过 `script_err_io!` 抛出 IO 错误。
