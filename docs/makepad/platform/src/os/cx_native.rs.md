# `cx_native.rs` — 原生平台 Cx 依赖加载与系统时间

## 文件定位

此文件为 `Cx`（Makepad 核心上下文）实现了两个依赖特定 OS 系统调用的方法：资源文件的磁盘加载和获取 Unix 时间戳。这些方法在 WebAssembly 目标上不可用，因此通过 `mod.rs` 中的条件编译（仅原生平台）控制是否编译。

---

## `EventFlow` 枚举

```rust
#[derive(PartialEq, Eq, Clone, Copy, Debug)]
pub enum EventFlow {
    Poll,
    Wait,
    Exit,
}
```

- **定义**：事件循环返回值的三种状态。
- **`Poll`**：事件循环应立即返回，不需要阻塞等待新事件（有事件待处理）。
- **`Wait`**：事件循环可以阻塞等待新事件（无待处理事件）。
- **`Exit`**：事件循环应退出（收到终止信号或窗口关闭）。
- **实现逻辑**：
  1. 平台层的事件循环驱动函数返回此枚举值。
  2. 上层循环根据返回值决定下次迭代行为：`Poll` → 立即重入；`Wait` → 阻塞直到新事件到达；`Exit` → 终止循环并清理资源。
  3. `PartialEq + Eq` 使事件循环可以比较返回状态进行分支。
  4. `Clone + Copy` 使枚举值可以轻松在函数调用间传递，无需所有权管理。

---

## `Cx::native_load_dependencies(&mut self)`

```rust
pub fn native_load_dependencies(&mut self) {
    for (path, dep) in &mut self.dependencies {
        if let Ok(mut file_handle) = File::open(path) {
            let mut buffer = Vec::<u8>::new();
            if file_handle.read_to_end(&mut buffer).is_ok() {
                dep.data = Some(Ok(Rc::new(buffer)));
            } else {
                dep.data = Some(Err("read_to_end failed".to_string()));
            }
        } else {
            println!("Could not load resource {}", path);
            dep.data = Some(Err("File! open failed".to_string()));
        }
    }
}
```

- **职责**：从文件系统加载所有注册的外部资源依赖到内存。
- **实现逻辑**：
  1. 遍历 `self.dependencies` 中所有已注册的依赖项，每个依赖项包含一个文件路径 `path` 和一个 `Dependency` 结构体。
  2. 对每个路径调用 `File::open(path)` 尝试打开文件。如果打开失败，打印错误信息 `"Could not load resource {path}"`，并将依赖数据设置为 `Err("File! open failed")` 标记加载失败。
  3. 如果文件打开成功，创建一个空的 `Vec<u8>` 缓冲区，调用 `file_handle.read_to_end(&mut buffer)` 将整个文件内容读入缓冲区。
  4. 如果读取成功，将数据包装为 `Ok(Rc::new(buffer))` 存入 `dep.data`。使用 `Rc`（引用计数）使得同一份资源数据可以在多个地方共享而无需复制。
  5. 如果读取失败（例如文件损坏或权限不足），将 `dep.data` 设置为 `Err("read_to_end failed")`。
  6. 调用完成后，`self.dependencies` 中每个依赖项的 `data` 字段都有值（`Some(...)`），后续代码通过检查 `data` 是否为 `Ok` 来确定资源是否可用。
- **设计要点**：
  - 此方法在应用启动时调用一次，将所有静态资源（着色器、字体、图像等）预加载到内存中。
  - 失败不会 panic，而是记录错误并标记依赖为不可用，使应用可以在资源缺失时降级运行而不是崩溃。
  - 打印到 stdout 而非使用日志系统，因为日志系统本身可能依赖这些资源。

---

## `Cx::time_now() -> f64`

```rust
pub fn time_now() -> f64 {
    if let Ok(elapsed) = SystemTime::now().duration_since(SystemTime::UNIX_EPOCH) {
        return elapsed.as_secs_f64();
    }
    return 0.0;
}
```

- **职责**：获取当前系统时间（Unix 纪元以来的秒数，浮点数精度）。
- **实现逻辑**：
  1. 调用 `SystemTime::now()` 获取当前系统时间。
  2. 调用 `.duration_since(SystemTime::UNIX_EPOCH)` 计算从 1970-01-01 UTC 到现在的时间间隔。
  3. 如果计算成功（系统时间不早于 Unix 纪元），调用 `.as_secs_f64()` 将 `Duration` 转换为以秒为单位的 `f64`，返回包含小数部分的精确时间。
  4. 如果失败（例如系统时钟被设置为 Unix 纪元之前），返回 `0.0` 作为安全默认值。
  5. 此方法是关联函数（`impl Cx` 上的 `fn`，非 `&self`），不需要 `Cx` 实例即可调用，适合在没有任何上下文时获取时间戳。
- **设计要点**：
  - 使用 `f64` 作为时间表示：秒为单位，小数部分提供亚毫秒精度，足以满足动画和帧计时需求。
  - 返回 `0.0` 的降级策略确保即使系统时钟异常，应用不会 panic，但动画/计时行为可能会异常。
