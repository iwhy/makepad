# `overlay.rs` — 覆盖层系统

**文件路径**: `draw/src/overlay.rs` (75 行)  
**核心作用**: 实现 UI 覆盖层（overlay）机制，管理悬浮层、弹出层、工具提示等需要在主布局之上独立绘制的 UI 元素。每个覆盖层拥有独立的 `DrawList`，在主渲染完成后附加到父级 DrawList 上。

---

## 一、`Overlay` 结构体（第 9–14 行）

```rust
#[derive(Debug, Script, ScriptHook)]
pub struct Overlay {
    #[new]
    pub draw_list: DrawList,
}
```

**字段**:
- `draw_list: DrawList` — 覆盖层专用的独立绘制列表。`#[new]` 属性表示在构造时自动创建。

覆盖层通过 `DrawList` 实现渲染隔离——覆盖层内的所有绘制命令写入独立的 `DrawList`，最后作为子列表（sub list）附加到父级渲染流。

**注意**: 原有的 `sweep_lock`（基于 `Rc<RefCell<Area>>` 的鼠标事件锁定机制）已被注释掉（第 14 行、第 19–27 行）。这表明覆盖层的事件路由机制可能已重构，或有新的实现方案。

---

## 二、事件处理

### `handle_event(&self, _cx: &Cx, _event: &Event)`（第 18–28 行）

**当前实现**: 空函数体。原本的 `sweep_lock` 机制（第 19–27 行注释）用于在鼠标事件发生时锁定覆盖层的响应区域，确保覆盖层在鼠标移动/点击/滚动时优先获得事件。

被注释的逻辑：
- `MouseMove` / `MouseDown` / `Scroll` 事件将 `Area` 设置到 `event.sweep_lock`
- 这使得事件系统可以判断鼠标是否在覆盖层区域内

---

## 三、覆盖层生命周期

### `begin(&self, cx: &mut Cx2d)`（第 30–35 行）

开始覆盖层绘制：
1. 设置 `cx.overlay_id = Some(self.draw_list.id())` — 标记当前覆盖层的 DrawList ID
2. 清除 `cx.overlay_pass_id = None` — 不绑定特定 pass

此后所有在 `Cx2d` 上发出的绘制命令都会写入覆盖层的 DrawList，而非主布局的 DrawList。

### `begin_for_pass(&self, cx: &mut Cx2d, pass_id: DrawPassId)`（第 37–42 行）

为特定渲染 pass 开始覆盖层：
1. 设置 `cx.overlay_id = Some(self.draw_list.id())`
2. 设置 `cx.overlay_pass_id = Some(pass_id)` — 绑定到指定 pass

这让覆盖层可以绑定到特定的渲染 pass（如后处理 pass），在多 pass 渲染管线中精确控制覆盖层的渲染时机。

### `end(&self, cx: &mut Cx2d)`（第 44–74 行）

结束覆盖层绘制并执行清理：

**第 45–46 行**: 清除 `cx.overlay_id` 和 `cx.overlay_pass_id`，恢复主布局的绘制上下文。

**第 47–49 行**: 将覆盖层的 DrawList 作为子列表附加到父级 DrawList：
```rust
let parent_id = cx.draw_list_stack.last().cloned().unwrap();
let redraw_id = cx.redraw_id;
cx.draw_lists[parent_id].append_sub_list(redraw_id, self.draw_list.id());
```
父级 DrawList 来自 `cx.draw_list_stack` 栈顶，`redraw_id` 确保在增量重绘时正确匹配。

**第 51–73 行 — 过期覆盖层清理**：
遍历覆盖层 DrawList 的所有绘制项，检查每个子列表：

1. **安全索引访问**：使用 `checked_index`（第 56、59 行）替代直接索引，避免因 DrawList 回收导致的越界访问
2. **Redraw ID 匹配检查**（第 60 行）：如果子 DrawList 的 `redraw_id` 与父 DrawList 不匹配，说明该子列表在上一次帧中已过期，需要清理
3. **回收保护**（第 64–71 行）：如果父 DrawList 或子 DrawList 已被回收（`checked_index` 返回 `None`），同样清理该子列表

这个清理机制确保覆盖层不会累积过期内容。

---

## 四、使用模式

```rust
// 创建覆盖层
let overlay = Overlay { draw_list: DrawList::new() };

// 开始覆盖层绘制
overlay.begin(&mut cx);

// 在覆盖层中绘制内容（所有 draw 命令写入 overlay.draw_list）
// ...

// 结束覆盖层
overlay.end(&mut cx);
// 此时 overlay.draw_list 作为子列表附加到父级 DrawList
```

---

## 五、关键设计

1. **DrawList 隔离**：每个覆盖层拥有独立的 DrawList，避免污染主渲染流
2. **Pass 绑定**：通过 `begin_for_pass` 支持多 pass 渲染管线
3. **自动过期清理**：通过 redraw_id 匹配机制自动清理失效的子列表
4. **安全访问**：使用 `checked_index` 防止 DrawList 回收后的越界错误
5. **可脚本化**：`Overlay` 实现了 `Script` 和 `ScriptHook`，可在 DSL 中声明使用
