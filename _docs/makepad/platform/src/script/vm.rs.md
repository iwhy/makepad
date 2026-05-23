# `platform/src/script/vm.rs` — ScriptVm ↔ Cx 双向访问桥

## 文件职责

该文件定义了 `ScriptVmCx` trait，这是 Splash VM 脚本引擎与 Makepad 平台上下文 `Cx` 之间的**双向桥接接口**。它使得 `ScriptVm` 在脚本执行过程中可以获取和操作 `Cx` 的状态，同时实现了 `bx`（VM 的字节码/堆状态）与 `cx.script_vm` 之间的安全指针交换。

---

## `ScriptVmCx` trait

```rust
pub trait ScriptVmCx {
    fn cx_mut(&mut self) -> &mut Cx;
    fn cx(&mut self) -> &Cx;
    fn with_cx<R, F: FnOnce(&Cx) -> R>(&mut self, f: F) -> R;
    fn with_cx_mut<R, F: FnOnce(&mut Cx) -> R>(&mut self, f: F) -> R;
}
```

四个方法分为两组：
- **直接引用**：`cx()` 和 `cx_mut()` 快速获取 `Cx` 引用（无状态交换）
- **闭包桥接**：`with_cx()` 和 `with_cx_mut()` 执行状态交换后的操作

---

## `ScriptVm<'a>` 实现

### `cx_mut()`

```rust
fn cx_mut(&mut self) -> &mut Cx {
    self.host.downcast_mut().unwrap()
}
```

- 通过 `ScriptVm` 的泛型 `host: Box<dyn Any>` 字段直接向下转型为 `&mut Cx`
- 这是最轻量的访问方式——不进行 `bx` 交换，仅解引用
- 适用于不需要访问脚本 VM 状态的纯平台操作（如读取 `os_type`）

### `cx()`

与 `cx_mut()` 相同模式，返回不可变引用：
```rust
self.host.downcast_ref().unwrap()
```

注意：两个方法都接受 `&mut self`，因为 `ScriptVm` 的所有方法都需要可变引用（VM 状态本身是可变的），但 `cx()` 返回 `&Cx` 提供只读访问。

### `with_cx(f)` 和 `with_cx_mut(f)`

这两个方法执行核心的**bx 交换协议**：

```rust
fn with_cx<R, F: FnOnce(&Cx) -> R>(&mut self, f: F) -> R {
    let saved_thread_id = self.bx.threads.current();
    let cx: &mut Cx = self.host.downcast_mut().unwrap();
    let bx = std::mem::replace(&mut self.bx, Box::new(ScriptVmBase::empty()));
    cx.script_vm = Some(bx);
    let r = f(cx);
    self.bx = cx.script_vm.take().unwrap();
    self.bx.threads.set_current(saved_thread_id);
    r
}
```

**为什么需要 bx 交换？**

`ScriptVm` 的正常工作状态是 `bx` 字段持有 VM 的堆（`ScriptHeap`）、字节码（`ScriptCode`）和线程调度器。当脚本代码需要调用平台方法（如 `cx.start_timeout()` 创建定时器）时，平台方法可能需要访问脚本状态来完成操作。

然而 `Cx` 上存储了平台定时器列表等状态，这些状态在 `ScriptVm` 创建之前和销毁之后都需要存在。通过将 `bx` 暂时存放到 `cx.script_vm`，平台方法回调可以通过 `cx` 上的 `with_vm()` 方法重新获取 `ScriptVm`，实现**循环引用中的安全通行**。

**交换协议细节：**

1. **保存线程 ID**：记录当前脚本线程 ID，以便交换后恢复上下文
2. **获取 `Cx` 引用**：从 `host` 向下转型
3. **交换 `bx`**：用 `std::mem::replace` 将 `self.bx` 替换为一个空的 `ScriptVmBase`，将原 `bx` 存入 `cx.script_vm = Some(bx)`
4. **执行闭包**：传递 `cx` 给闭包，此时 `cx.script_vm` 持有完整的 VM 状态
5. **恢复 `bx`**：执行 `cx.script_vm.take()` 将 `bx` 取出，放回 `self.bx`
6. **恢复线程 ID**：重置当前线程上下文

`with_cx_mut` 与 `with_cx` 的区别仅在于闭包参数类型：`&mut Cx` vs `&Cx`。

---

## `&mut dyn Any` 实现

```rust
impl ScriptVmCx for &mut dyn Any {
    // cx/cx_mut: 直接 downcast_mut/downcast_ref
    // with_cx/with_cx_mut: 没有 bx 交换，仅调用闭包
}
```

这个实现是 `makepad_script_std` 中的 `ScriptVmBase` 的 `host` 字段用的。当 `ScriptVmBase` 通过 `with_vm` 被包装为临时 `ScriptVm` 时，它的 `host` 就是 `&mut dyn Any`。

因为 `ScriptVmBase` 不包含 `bx` 字段（它是 `ScriptVm` 对 `ScriptVmBase` 的包装），所以 `with_cx` 不需要执行 bx 交换——它只是简单地将 `host` 向下转型后调用闭包。

这种模式使得 `makepad_script_std` 通用代码可以通过 `ScriptVmCx` trait 获取平台上下文，而不需要知道具体的 `Cx` 类型——它只知道它是一个 `dyn Any`。

---

## 设计要点

1. **双 trait 实现**：`ScriptVm<'a>` 的实现包含 bx 交换（用于完整的 VM 操作），`&mut dyn Any` 的实现仅做向下转型（用于 vm_std 内部的轻量访问）

2. **线程上下文保护**：交换期间保存并恢复线程 ID，防止嵌套调用导致线程上下文丢失

3. **零成本抽象**：`cx()` 和 `cx_mut()` 是简单的指针解引用和转型，无额外开销

4. **安全性**：虽然使用了 `std::mem::replace` 和 `unwrap`，但整个协议保证 `cx.script_vm` 在被 take 之前一定是 `Some`（因为刚刚被放入），且交换返回后 `self.bx` 一定非空
