# `drag_drop.rs` — 拖放事件系统

## 概述

该文件实现了 Makepad 的拖放（Drag & Drop）事件处理系统。它定义了拖放操作的事件结构、状态跟踪机制，以及 `Event::drag_hits()` 系列方法，用于将平台拖放事件转换为 widget 可处理的 `DragHit` 枚举。

---

## 基础事件类型

### `DragEvent`
拖拽过程中的事件：
- `modifiers: KeyModifiers` — 修饰键状态
- `handled: Arc<Mutex<bool>>` — 是否已被处理（使用 `Arc<Mutex<>>` 实现跨线程的共享可变性）
- `abs: Vec2d` — 当前位置
- `items: Arc<Vec<DragItem>>` — 拖拽携带的数据项
- `response: Arc<Mutex<DragResponse>>` — 拖拽操作响应类型

### `DropEvent`
拖放释放事件：
- 同 `DragEvent`，但不含 `response` 字段

---

## 命中结果类型

### `DragHitEvent`
拖拽命中的详细信息：
- `state: DragState` — 拖拽状态（In/Over/Out）
- `rect: Rect` — 目标区域
- `items`, `modifiers`, `abs`, `response` — 同 `DragEvent`

### `DropHitEvent`
释放命中的详细信息：
- 不含 `state` 和 `response` 字段

---

## 辅助枚举

### `DragState`
- `In` — 拖拽首次进入区域
- `Over` — 拖拽在区域内移动
- `Out` — 拖拽离开区域

### `DragResponse`
- `None` — 无响应
- `Copy` — 允许复制
- `Link` — 允许创建链接
- `Move` — 允许移动

### `DragItem`
拖拽数据项的类型：
- `FilePath { path: String, internal_id: Option<LiveId> }` — 文件路径，可携带内部 ID
- `String { value: String, internal_id: Option<LiveId> }` — 文本字符串

---

## `CxDragDrop` — 拖放状态跟踪

轻量级状态管理器：
- `drag_area: Area` — 当前拖拽所在的区域
- `next_drag_area: Area` — 下一帧的拖拽区域（暂存）

### `CxDragDrop::cycle_drag()`
将 `next_drag_area` 提交为 `drag_area`，并将 `next_drag_area` 重置为 `Area::Empty`。

### `CxDragDrop::update_area(old_area, new_area)`
区域重映射时更新 `drag_area` 引用。

---

## `Event` 的拖放命中测试方法

### `Event::drag_hits(cx, area) -> DragHit`
使用默认选项的拖放命中测试，委托给 `drag_hits_with_options`。

### `Event::drag_hits_with_options(cx, area, options) -> DragHit`
核心拖放命中测试方法，按事件类型处理：

#### `Drag` 事件
区分两种情况：

1. **`area` 已标记为当前拖拽区域**（`area == cx.drag_drop.drag_area`）：
   - 检查事件是否未处理且位置在区域内
   - 如果命中：标记已处理，设置 `next_drag_area = area`，返回 `DragHit::Drag { state: Over }`
   - 如果未命中：返回 `DragHit::Drag { state: Out }`（离开信号）

2. **`area` 不是当前拖拽区域**：
   - 检查事件是否未处理且位置在区域内
   - 如果命中：标记已处理，设置 `next_drag_area = area`，返回 `DragHit::Drag { state: In }`（进入信号）
   - 如果未命中：返回 `DragHit::NoHit`

这种设计确保拖拽进入/离开的边界检测：首次进入时返回 `In`，后续保持在内部的移动返回 `Over`，离开时返回 `Out`。

#### `Drop` 事件
- 检查事件是否未处理且位置在区域内
- 如果命中：标记已处理，重置 `next_drag_area = Area::Empty`，返回 `DragHit::Drop`
- 如果未命中：返回 `DragHit::NoHit`

#### 其他事件
- 一律返回 `DragHit::NoHit`
