# `uniform_buffer.rs` — 统一缓冲区分配与管理

## 概述

`uniform_buffer.rs` 定义了 Makepad 框架中 GPU 统一缓冲区（Uniform Buffer）的轻量分配和管理系统。统一缓冲区用于向 GPU 着色器传递常量数据（如变换矩阵、材质参数等）。

文件虽小（103 行），但完整实现了统一缓冲区的分配、复用、数据读写和脚本集成。

---

## 核心类型

### `UniformBuffer`（第 9-10 行）

```rust
pub struct UniformBuffer(Rc<PoolId>);
```

与 `Texture` 类似的引用计数句柄。通过 `Rc<PoolId>` 实现安全的 Clone/Drop。内部 `PoolId` 包含 `id`（槽索引）和 `generation`（代数），用于 `CxUniformBufferPool` 中的定位和退役检测。

### `UniformBufferId`（第 12-13 行）

```rust
pub struct UniformBufferId(pub(crate) usize, u64);
```

- `usize`：池槽索引
- `u64`：槽位代数，用于悬空检测

---

## `UniformBuffer` 方法

| 方法 | 说明 |
|------|------|
| `new(cx)` | 从 `cx.uniform_buffers.alloc()` 创建一个新的 UniformBuffer |
| `uniform_buffer_id()` | 返回 `(池索引, 代数)` 格式的唯一标识符 |
| `clear(cx)` | 清空缓冲区数据：`data.clear()` |
| `set_bytes(cx, data)` | 设置原始字节数据。先清空已有的 data，再扩展填充新数据 |
| `set_struct::<T: Copy>(cx, value)` | 写入一个 Copy 类型结构体。通过 `std::slice::from_raw_parts` 将结构体的内存表示转为 `&[u8]` 后调用 `set_bytes` |

**实现细节：** `set_struct` 使用 `unsafe` 代码将任意 `Copy` 类型的内存布局直接解释为字节切片。这是 GPU 驱动中的常见模式——CPU 端的结构体定义（如 `mat4x4`、`vec4`）与 GPU 端的 uniform block 布局对应。调用者必须确保结构体的内存布局（对齐、padding）符合着色器的预期布局。

| 方法 | 说明 |
|------|------|
| `set_struct_slice::<T: Copy>(cx, values)` | 写入结构体数组。将连续内存的切片转为 `&[u8]` 后调用 `set_bytes` |

---

## `CxUniformBufferPool`——统一缓冲区池（第 57-70 行）

```rust
pub struct CxUniformBufferPool(pub(crate) IdPool<CxUniformBuffer>);
```

### `alloc()`——分配缓冲区

**实现逻辑：**
1. 调用 `IdPool::alloc_with_reuse_filter(|_| true, CxUniformBuffer::default())`。复用过滤始终返回 `true`，意味着任何空闲槽都可复用
2. 如果复用了旧槽（`previous_item` 存在），将旧槽的 OS 资源（`CxOsUniformBuffer`）转移到新槽，实现 GPU 资源的无损复用
3. 返回 `UniformBuffer(Rc::new(new_id))`

### Index/IndexMut（第 72-97 行）

通过 `UniformBufferId` 访问：
1. 从 `pool[index.0]` 获取槽位
2. 验证 `generation == index.1`，不匹配时输出错误日志（悬空引用检测）
3. 返回/返回可变 `&CxUniformBuffer`

---

## `CxUniformBuffer`——运行时数据（第 99-103 行）

```rust
pub struct CxUniformBuffer {
    pub data: Vec<u8>,        // 原始字节数据
    pub os: CxOsUniformBuffer, // 平台 GPU 统一缓冲区资源
}
```

- `data`：CPU 端存储的统一缓冲区字节。通过 `set_bytes` / `set_struct` 写入
- `os`：平台层的 GPU 缓冲区资源（OpenGL UBO / Vulkan buffer / Metal buffer），由 OS 层在渲染时上传到 GPU

---

## Script 集成

`UniformBuffer` 实现了 `ScriptHook`、`ScriptApply`、`ScriptNew`：
- `ScriptNew::script_new(vm)` → `UniformBuffer::new(vm.cx_mut())`

这使得 Lua/Script 层可以直接创建和操作 UniformBuffer。

---

## 总结

`uniform_buffer.rs` 提供了一个简洁、安全的统一缓冲区管理方案。通过 `IdPool` 的复用机制，GPU 缓冲区资源在频繁创建/销毁的过程中可以被无损重用（保留 OS 层的 GPU buffer 对象）。`set_struct` 和 `set_struct_slice` 提供了类型安全的 Rust 结构体到 GPU uniform block 的转换，避免了手动的序列化代码。
