# `defer_with_redraw.rs` — 延迟执行后自动重绘扩展

## 概述

为 `UiRunner` 提供 `defer_with_redraw` 扩展方法，在标准的延迟执行闭包末尾自动调用 `widget.redraw(cx)`，适合需要在下一帧执行逻辑后立即触发界面重绘的场景。

---

## 类型定义

### `trait DeferWithRedraw<T: 'static>`

```rust
pub trait DeferWithRedraw<T: 'static> {
    fn defer_with_redraw(self, f: impl DeferCallback<T>);
}
```

**功能**：扩展 trait，为 `UiRunner<T>` 增加一个会在闭包执行后自动重绘的推迟执行方法。

- `T` 为 `UiRunner` 持有的 widget 类型泛型参数，约束为 `'static`。
- `defer_with_redraw` 接受一个实现了 `DeferCallback<T>` 的闭包 `f`，闭包接收 `(widget, cx, scope)` 三个参数。
- 返回值与标准的 `defer` 方法一致，无返回值。

---

## Trait 实现

### `impl<W: Widget + 'static> DeferWithRedraw<W> for UiRunner<W>`

```rust
fn defer_with_redraw(self, f: impl DeferCallback<W>) {
    self.defer(|widget, cx, scope| {
        f(widget, cx, scope);
        widget.redraw(cx);
    });
}
```

**实现逻辑**：

1. **委托标准 `defer`**：直接调用 `UiRunner::defer` 将任务排入延迟队列。`self` 在此处消费了 `UiRunner` 的所有权，与标准 `defer` 的签名一致。

2. **包装闭包**：将用户传入的闭包 `f` 包裹在一个新闭包中。新闭包首先执行 `f(widget, cx, scope)`，让用户的自定义逻辑先运行。

3. **自动重绘**：在用户闭包执行完毕后，立即调用 `widget.redraw(cx)`。这会向事件循环发送重绘请求，确保在当前帧的图形处理阶段重新绘制该 widget。

4. **设计意图**：标准的 `defer` 机制仅在下一帧执行闭包，但不会主动触发绘制。如果 defer 中的逻辑修改了 widget 的状态（例如更新文本、颜色），界面不会立即响应。「延迟 + 重绘」的组合省去了手动调用 `redraw` 的步骤，减少了模板代码。

---

## 用途与场景

- **异步状态更新**：在事件处理中 defer 一个状态变更，并确保界面在下一帧刷新。
- **文本输入后更新**：`TextInput` 等 widget 在处理完输入后常用 defer 来同步布局，配合 `defer_with_redraw` 可同时触发重绘。
- **避免递归重绘**：如果在事件处理中直接调用 `redraw` 可能导致递归，defer 可将重绘推迟到当前事件循环结束后安全执行。
